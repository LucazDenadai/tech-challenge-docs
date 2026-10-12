# CARD-39a — Orçamento e aprovação no Billing

**Tipo:** Implementação / Domínio
**Status:** To Do — escopo definido em 2026-10-12
**Depende de:** CARD-35, CARD-36
**Bloqueia:** CARD-39b, CARD-40
**Repositório alvo:** `tech-challenge-billing` ([ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md))
**Decisão arquitetural:** ADR-014, [ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md), [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md), [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md) e [ADR-018](../../adr/ADR-018-contratos-assincronos-asyncapi-fase4.md); catálogo de contratos do CARD-36

---

## Contexto

O enunciado inclui geração e envio de orçamento para aprovação. Billing deve ser responsável pelo ciclo de orçamento e por manter a decisão financeira associada à ordem, enquanto OS mantém apenas o estado geral e Operações só recebe uma autorização de fluxo válida.

## Ponto de partida (2026-10-12)

- `tech-challenge-billing` ainda não existe. O repositório nasce neste card, público, com a mesma estrutura de `tech-challenge-os` e `tech-challenge-operacoes`: Domain, Application, Infrastructure e API, testes unitários e de integração (Testcontainers).
- O contrato não muda. O `QuoteRequested.v1` já traz `clienteId`, itens com preço (`PricedItem`), moeda e `priceSnapshotAtUtc`. Os eventos de saída usam `QuoteRef` (`quoteId`, `quoteVersion`) e `QuoteDecision` (`decidedAtUtc`).
- O token do cliente emitido pela Lambda (RFC-003, ADR-013) tem `role=Cliente` e a claim `cliente_id`, já usada pelo OS em `AcompanhamentoOSController`.
- Inbox, outbox, despachante, consumidor com DLQ e cópia da spec AsyncAPI seguem o padrão de Operações (CARD-38a). O código é copiado, não compartilhado por biblioteca, para manter os repositórios independentes.

## Decisões de implementação (2026-10-12)

- **Quem aprova:** só o cliente dono da OS. A decisão exige token com `role=Cliente` e `cliente_id` igual ao `ClienteId` do orçamento. Token de funcionário (`Admin`, `Atendente`, `Mecanico`) recebe `403`, mesmo para `Admin`. Um funcionário que traz o próprio veículo é tratado como cliente: entra pelo CPF na Lambda e decide com o token de cliente. Orçamento de outro cliente responde `404`, como no acompanhamento do OS.
- **Agregado:** `Orcamento` (`Id` = `quoteId`, `OsId` único, `ClienteId`, `FilialId`, `CorrelationId`) com as versões. Cada `VersaoOrcamento` tem `Numero` (= `quoteVersion`), itens (`ItemId`, `Tipo`, `Descricao`, `Quantidade`, `PrecoUnitario`), `Moeda`, `ValorTotal`, `PrecoSnapshotEm`, `ExpiraEm` e `Status`. A decisão vale só para a versão vigente (a mais recente).
- **Estados da versão:** `AguardandoDecisao` → `Aprovada`, `Recusada` ou `Expirada`. `Substituida` quando chega uma versão nova antes da decisão. Os quatro últimos são terminais.
- **Versões:** um `QuoteRequested` com nova `idempotencyKey` para uma OS com versão `AguardandoDecisao` cria a versão seguinte e marca a anterior como `Substituida`. Depois de uma decisão, novo pedido para a mesma OS gera `QuoteRejected` (revisão após aprovação fora do MVP, ADR-017). O fluxo normal da Saga cria só a versão 1; as versões existem para a regra de "versão desatualizada" do contrato.
- **Cálculo:** `ValorTotal` = soma de `Quantidade × PrecoUnitario`, em `decimal` com 2 casas. Item sem preço, preço não positivo, lista vazia ou moeda diferente de `BRL` geram `QuoteRejected` com motivo, sem criar orçamento. Billing não consulta Operações.
- **Validade:** `ExpiraEm` = publicação + `Orcamento:ValidadeHoras` (padrão 24). Um serviço em segundo plano procura versões `AguardandoDecisao` vencidas, marca `Expirada` e publica `QuoteExpired`. Uma decisão que chega depois de `ExpiraEm` responde `410` e grava a expiração na mesma transação, se o serviço ainda não gravou. `QuoteExpired` sai uma única vez.
- **Histórico:** `DecisaoOrcamento` (`Versao`, `Resultado`, `ClienteId`, `DecididaEm`, `Motivo` opcional na recusa) e as mudanças de estado ficam gravadas. Tentativa negada (`403`, `404`, `409`, `410`) não muda estado nem histórico; só vai para o log.
- **Idempotência da decisão:** repetir a mesma decisão para a mesma versão responde `200` com o resultado gravado, sem segundo evento. A decisão oposta sobre versão já decidida, ou decisão sobre versão `Substituida`, responde `409`.
- **Concorrência:** otimista pelo `xmin` do PostgreSQL, como o saldo em Operações. Duas decisões simultâneas: uma grava; a outra relê e recebe `200` (mesma decisão) ou `409` (decisão oposta).
- **API:** tudo sob `/billing/orcamentos`.

  | Rota | Quem | Para quê |
  |---|---|---|
  | `GET /billing/orcamentos/meus` | `Cliente` | Orçamentos do `cliente_id` do token, com a versão vigente. O cliente não precisa conhecer IDs internos. |
  | `GET /billing/orcamentos/{quoteId}` | `Cliente` dono, `Admin`, `Atendente` | Detalhe com versões e histórico. |
  | `GET /billing/orcamentos/os/{osId}` | `Admin`, `Atendente` | Orçamento de uma OS, para o atendimento informar o cliente. |
  | `POST /billing/orcamentos/{quoteId}/aprovar` | `Cliente` dono | Corpo `{ "versao": n }`. `200`, `403`, `404`, `409` ou `410`. |
  | `POST /billing/orcamentos/{quoteId}/recusar` | `Cliente` dono | Corpo `{ "versao": n, "motivo": "..." }`. Mesmas respostas. |

- **Mensageria:** consome `QuoteRequested`; publica `QuoteReady`, `QuoteRejected`, `QuoteApproved`, `QuoteDeclined` e `QuoteExpired`. Inbox, efeito e outbox na mesma transação. Falha de processamento vai para a DLQ com motivo (ADR-018). Os canais de pagamento e estorno entram no CARD-39b.
- **Porta para o 39b:** `IOrcamentoParaPagamento` responde se uma `(quoteId, versao)` está `Aprovada` e devolve valor, moeda e `ClienteId`. É a única entrada do Pagamento no Orçamento.
- **Dados pessoais:** Billing guarda só `ClienteId`. Nome, CPF e e-mail do pagador ficam para o desenho do CARD-39b (ADR-015).
- **Local e testes:** `docker-compose.yml` com PostgreSQL (porta `5434` no host, para não colidir com OS e Operações), RabbitMQ e Jaeger. JWT com o mesmo issuer e chave dos outros serviços. Os testes de integração sobem PostgreSQL e RabbitMQ pelo Testcontainers.
- **Fora deste card:** pagamento, Mercado Pago e webhook (CARD-39b); consumo dos eventos de orçamento pelo OS (CARD-40); imagem, deploy e CI (CARD-41).

## Critérios de aceite

- [ ] Orçamento tem versão, valor/moeda, itens referenciados, validade, filial e estado de aprovação.
- [ ] Criação não lê tabelas de OS/Operações; os dados necessários chegam via contrato aprovado. O valor é calculado a partir do snapshot de preços do diagnóstico, sem consulta a Operações (ADR-017).
- [ ] Aprovação/rejeição autenticada e repetida é idempotente e mantém histórico/auditoria.
- [ ] Aprovação se aplica à versão exata apresentada ao cliente; decisão para versão antiga é rejeitada se o orçamento foi alterado.
- [ ] Somente orçamento vigente e aprovado pode iniciar pagamento; orçamento pendente, rejeitado, cancelado ou expirado não pode gerar cobrança nem liberar execução.
- [ ] Decisão sem autorização válida não altera o estado do orçamento nem publica evento de aprovação/rejeição.
- [ ] Mudança de orçamento após aprovação é rejeitada no MVP; revisão de diagnóstico/orçamento fica fora do escopo inicial (ADR-017).
- [ ] Aprovação/rejeição é exposta pelo Billing via REST síncrono (API Gateway, JWT existente), respondendo sucesso, versão desatualizada (`409`) ou orçamento expirado (`410`); o evento de decisão é publicado via outbox (ADR-017).
- [ ] Resultado é publicado via evento conforme CARD-36 e não altera banco OS diretamente.
- [ ] Testes cobrem validade, aprovação, rejeição, expiração e concorrência de decisão.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Aprovar orçamento de uma OS

  Cenário: Cliente aprova orçamento válido
    Dado que existe um orçamento pendente e não expirado
    Quando o cliente autorizado aprova a versão apresentada
    Então Billing registra a aprovação dessa versão
    E publica o evento de aprovação uma única vez

  Cenário: Aprovação de orçamento expirado
    Dado que o orçamento expirou
    Quando o cliente tenta aprová-lo
    Então Billing rejeita a decisão
    E a OS não é liberada para execução

  Cenário: Rejeitar decisão sobre uma versão antiga
    Dado que o cliente recebeu a versão 1 do orçamento
    E Billing publicou uma versão 2 antes da resposta do cliente
    Quando chega uma aprovação referente à versão 1
    Então Billing rejeita a decisão como desatualizada
    E mantém a versão 2 aguardando decisão
    E não publica evento de orçamento aprovado

  Cenário: Rejeitar decisão de usuário sem autorização
    Dado que o orçamento está pendente
    E o solicitante não pode decidir por essa OS
    Quando tenta aprovar ou rejeitar o orçamento
    Então Billing rejeita a solicitação
    E mantém estado e histórico de decisão inalterados

  Cenário: Não criar cobrança para orçamento rejeitado
    Dado que o cliente rejeitou o orçamento
    Quando o fluxo tenta solicitar um pagamento
    Então Billing não cria uma tentativa no provedor
    E publica ou retorna somente o resultado de rejeição previsto no contrato
    E Operações não recebe autorização para iniciar a execução
```

## Passos

1. Modelar orçamento e histórico como agregado de Billing.
2. Definir identidade/ autorização do aprovador sem transportar dados de pagamento sensíveis.
3. Implementar API e eventos, depois testes de integração.
4. Atualizar Swagger e collection com exemplos fictícios.

## Evidências

- Testes unitários/integração, OpenAPI e amostra de evento versionado.