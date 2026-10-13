# CARD-39b — Mercado Pago: cobrança e webhook

**Tipo:** Integração externa / Pagamentos
**Status:** Implementado — PR #2 mergeado em `tech-challenge-billing` (2026-10-13); validação em sandbox pendente (credenciais e URL pública) e evidência de CI pendente (CARD-41)
**Depende de:** CARD-36, CARD-39a
**Bloqueia:** CARD-40
**Repositório alvo:** `tech-challenge-billing` ([ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md))
**Decisão arquitetural:** [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md), [ADR-018](../../adr/ADR-018-contratos-assincronos-asyncapi-fase4.md) e [ADR-019](../../adr/ADR-019-integracao-mercado-pago-fase4.md); contratos do CARD-36

---

## Contexto

Pagamento é uma fronteira assíncrona: criação de preferência/intenção não confirma liquidação e o retorno do navegador não é fonte confiável. Billing deve integrar-se à API atual do Mercado Pago, validar notificações e reconciliar o estado consultando o provedor conforme documentação oficial selecionada. A integração começa em sandbox.

## Ponto de partida (2026-10-13)

- O CARD-39a deixou a porta `IOrcamentoParaPagamento`: responde se uma `(quoteId, versao)` está `Aprovada` e devolve valor, moeda, `ClienteId`, OS e filial. Inbox, outbox, consumidor com DLQ e despachante já existem em `tech-challenge-billing`.
- O AsyncAPI já tem os canais de pagamento e estorno. Billing consome `PaymentCreationRequested` e `PaymentRefundRequested`; publica `PaymentPending`, `PaymentApproved`, `PaymentDeclined`, `PaymentOutcomeUnknown`, `PaymentRefunded`, `PaymentRefundPending` e `PaymentRefundFailed`.
- O time ainda não tem credenciais do Mercado Pago. A implementação usa a porta do provedor com um provedor simulado; a validação em sandbox (passo 6) fica pendente até haver credenciais e uma URL pública para o webhook.

## Decisões de implementação (2026-10-13)

As decisões de integração (produto, autenticação, estados, idempotência, retry, dados do pagador) estão no [ADR-019](../../adr/ADR-019-integracao-mercado-pago-fase4.md). Resumo e decisões de código:

- **Produto:** Pix pela API de Pagamentos (`POST /v1/payments`), sem dados de cartão em Billing. `external_reference` = `paymentId`; expiração de 30 minutos (`MercadoPago:ExpiracaoMinutos`).
- **Contrato:** `PaymentCreationRequested.v1` ganha `payerEmail` opcional (mudança compatível, ADR-018). Billing manda para a DLQ o comando sem e-mail, para orçamento não aprovado ou com valor/moeda diferentes da versão aprovada: nenhum desses casos cria cobrança.
- **Agregado:** `Pagamento` (`Id` = `paymentId`, `OsId`, `FilialId`, `CorrelationId`, `ClienteId`, `QuoteId`, `QuoteVersion`, `Valor`, `Moeda`, `IdempotencyKey` único, `ReferenciaProvedor`, `CodigoPix`, `UrlPagamento`, `Status`, dados do estorno e contagem de falhas de reconciliação).
- **Estados:** `Solicitado` → `Pendente` → `Aprovado` ou `Recusado`; `Aprovado` → `EstornoPendente` → `Estornado` ou `EstornoFalhou`. Só avançam. Cada evento sai uma vez, na transição, pela outbox.
- **Chamada externa fora da transação:** o comando grava inbox e `Pagamento` em `Solicitado` em uma transação. A chamada ao provedor acontece depois do commit, e o resultado é gravado com o evento em outra transação. Se o processo cair entre as duas, a reconciliação refaz a criação com a mesma chave idempotente.
- **Webhook:** `POST /billing/pagamentos/webhook/mercadopago`, anônimo, com `x-signature` validada. Notificação válida dispara `GET /v1/payments/{id}`; o corpo não muda estado. Cada notificação autenticada é gravada com o resultado (`Aplicada`, `SemMudanca`, `Desconhecido`, `Divergente`, `ForaDoFluxo`, `FalhaTecnica`) para auditoria; assinatura inválida responde `401` e vai só para log e métrica.
- **Reconciliação:** serviço em segundo plano (`MercadoPago:IntervaloReconciliacaoSegundos`, padrão 60) refaz `Solicitado`, consulta `Pendente` e `EstornoPendente`. Depois de 10 falhas técnicas seguidas, marca `ReconciliacaoManual`.
- **Estorno:** total, com `X-Idempotency-Key = estorno-<paymentId>`. Pedido repetido republica o resultado gravado sem chamar o provedor. Estorno recusado pelo provedor ou falha técnica além do limite → `EstornoFalhou` e `PaymentRefundFailed` (a Saga vai para `ManualActionRequired`).
- **API para o cliente pagar:** `GET /billing/pagamentos/meus` (`Cliente`) e `GET /billing/pagamentos/os/{osId}` (`Admin`, `Atendente`) devolvem estado, código Pix e URL de pagamento.
- **Provedor:** `MercadoPago:Modo` = `Simulado` (padrão local e testes) ou `MercadoPago`. No modo `Simulado`, `POST /billing/simulador/pagamentos/{referencia}/status` muda o status no provedor em memória e reconcilia, para o BDD e o vídeo. A rota não existe no modo `MercadoPago`.
- **Observabilidade:** métricas `billing_mercadopago_chamadas` (operação, resultado), `billing_webhook_notificacoes` (resultado) e `billing_pagamentos_reconciliacao_manual`. Token, segredo e e-mail nunca entram em log.
- **Fora deste card:** o OS enviar `payerEmail` e consumir os eventos de pagamento (CARD-40); Secret do token e URL pública do webhook (CARD-41); cancelar Pix pendente quando a Saga é cancelada (o Pix expira sozinho em 30 minutos; ver pendências).

## Implementação (2026-10-13)

- **Repositório:** [tech-challenge-billing](https://github.com/LucazDenadai/tech-challenge-billing), PR #2, em 8 commits, um por passo: domínio, contratos, casos de uso, persistência, adapters do Mercado Pago, API, teste pelo broker e README/OpenAPI.
- **Chamada externa depois do commit:** o `ProcessarMensagemSagaUseCase` devolve o evento e uma ação pós-commit. Se a ação falhar, a mensagem não volta à fila (a inbox a trataria como duplicada); o pagamento fica com verificação agendada e a reconciliação refaz.
- **Mesma chave, duas chamadas:** um comando repetido que chega enquanto o primeiro ainda está em `Solicitado` também chama o provedor. É seguro: a chave idempotente é a mesma e o provedor devolve a mesma cobrança. O teste de concorrência verifica cobranças distintas no provedor, não chamadas.
- **Resposta a comando repetido:** a mesma `idempotencyKey` em outra mensagem republica o evento do estado atual (`PaymentPending`, `PaymentApproved` ou `PaymentDeclined`) com o `causationId` da nova mensagem. No estorno, republica `PaymentRefunded`, `PaymentRefundPending` ou `PaymentRefundFailed` sem chamar o provedor.
- **Assinatura:** o validador também aceita `ts` em milissegundos (valor grande demais para segundos), por segurança; a documentação mostra segundos. O teste compara o HMAC com um hash calculado fora do código (Python).
- **Provedor simulado:** singleton em memória; os dados somem ao reiniciar o serviço. A rota do simulador exige `Admin` e responde `404` fora do modo `Simulado`.
- **Migration:** `Pagamentos` cria as tabelas `Pagamentos` e `NotificacoesPagamento`. Índices únicos em `IdempotencyKey`, `(QuoteId, QuoteVersion)`, `ReferenciaProvedor` e `RequestId`.

### Pendências

- **Sandbox:** sem credenciais do Mercado Pago ainda. Falta rodar criação, consulta, webhook assinado e estorno contra o sandbox e registrar a evidência anonimizada (passo 6). Precisa de uma URL pública para o webhook (túnel local ou o deploy do CARD-41). Sem webhook, a reconciliação por consulta já funciona.
- **Estorno de pagamento não aprovado:** o comando vai para a DLQ com o motivo, sem chamar o provedor. O contrato não tem evento para "nada a estornar"; o OS só pede estorno depois de `PaymentApproved` (ADR-017). Se o CARD-40 precisar de resposta, o contrato ganha o evento.
- **Estorno em processamento sem prazo:** enquanto o provedor responde, um estorno `in_process` é consultado a cada verificação sem limite. A reconciliação dedicada de estorno pendente está como evolução no ADR-017.
- **Pix pendente quando a Saga cancela:** não há comando para cancelar a cobrança; o Pix expira sozinho em 30 minutos e vira `PaymentDeclined`.
- **Réplicas:** a reconciliação tolera réplicas (`xmin` e chave idempotente). A trava da outbox continua no CARD-41.

## Critérios de aceite

- [x] ADR/RFC de integração registra produto/API do Mercado Pago escolhido, autenticação, estados, idempotency key e política de retry/rate limit. *([ADR-019](../../adr/ADR-019-integracao-mercado-pago-fase4.md).)*
- [x] Criação de pagamento associa identificador interno, OS, orçamento, filial e identificador do provedor sem salvar dados de cartão.
- [x] Webhook verifica assinatura/autenticidade segundo a documentação vigente e valida o recurso consultando o provedor quando aplicável.
- [x] A chave idempotente impede cobrança duplicada em retry da mesma solicitação.
- [x] Webhook repetido, atrasado, inválido e de estado terminal não regressa estado nem duplica evento.
- [x] O pagamento consultado no provedor corresponde à tentativa interna, OS, orçamento, valor e moeda antes de qualquer confirmação. *(O `external_reference` identifica a tentativa, que guarda OS e orçamento; referência, valor e moeda são conferidos.)*
- [x] Estado de recusa do provedor é tratado como resultado de negócio; erro HTTP, timeout ou indisponibilidade permanece resultado técnico desconhecido e segue retry/reconciliação.
- [x] Webhook válido para pagamento desconhecido ou com associação divergente não altera estado e segue o tratamento de segurança/auditoria definido. *(Gravado em `NotificacoesPagamento` com o resultado e log de aviso.)*
- [x] Falha temporária do provedor gera retry limitado e observável; falha persistente é encaminhada para reconciliação/DLQ conforme decisão. *(Ação manual com log de erro e métrica, ADR-019.)*
- [x] Aprovação publicada só ocorre após confirmação autoritativa do pagamento.
- [x] Caminho de cancelamento/estorno ou recuperação manual está exercitado para falha posterior na Saga. *(Com o provedor simulado e o HTTP com respostas gravadas; no sandbox, pendente.)*
- [ ] Testes cobrem sandbox e cenários simulados sem expor tokens em log. *(Simulados e respostas gravadas cobertos; sandbox pendente de credenciais.)*

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Integrar pagamentos com Mercado Pago

  Cenário: Criar pagamento sem duplicar cobrança
    Dado que existe um orçamento aprovado
    Quando a mesma solicitação idempotente de pagamento é enviada duas vezes
    Então existe no máximo uma cobrança correspondente no provedor
    E Billing retorna a mesma referência de pagamento

  Cenário: Receber webhook de pagamento aprovado
    Dado que o provedor envia uma notificação válida de pagamento
    E a consulta ao provedor confirma o estado aprovado
    Quando Billing processa o webhook
    Então o estado financeiro é persistido como aprovado
    E um único evento de confirmação é publicado

  Cenário: Não aceitar webhook adulterado
    Dado que a assinatura da notificação é inválida
    Quando Billing recebe o webhook
    Então a notificação é rejeitada e auditada
    E o estado de pagamento permanece inalterado

  Cenário: Não confirmar tentativa com valor ou moeda divergentes
    Dado que um webhook autenticado referencia uma tentativa existente
    Quando Billing consulta o pagamento no Mercado Pago
    E o valor ou a moeda não correspondem ao orçamento aprovado
    Então Billing não marca o pagamento como aprovado
    E não publica evento de pagamento aprovado
    E registra a divergência sem expor dados sensíveis

  Cenário: Registrar recusa de pagamento sem liberar execução
    Dado que o Mercado Pago confirma que a tentativa foi recusada
    Quando Billing processa a notificação e reconcilia o estado
    Então a tentativa é registrada como recusada
    E a Saga recebe o evento de recusa definido no contrato
    E nenhum evento de pagamento aprovado é publicado

  Cenário: Não tratar indisponibilidade como recusa
    Dado que a consulta ao Mercado Pago falha por timeout ou indisponibilidade
    Quando Billing processa uma notificação ainda não reconciliada
    Então o pagamento permanece pendente ou com resultado desconhecido
    E Billing agenda retry/reconciliação conforme política
    E não publica evento de recusa nem de aprovação

  Cenário: Não regredir pagamento por notificação fora de ordem
    Dado que Billing já registrou o pagamento como aprovado
    Quando recebe uma notificação válida de estado pendente anterior
    Então mantém o pagamento aprovado
    E não publica uma nova transição incompatível

  Cenário: Colocar em investigação pagamento desconhecido ou associado a outra OS
    Dado que um webhook autenticado referencia um pagamento inexistente ou incompatível com a OS ou orçamento
    Quando Billing tenta reconciliar o pagamento
    Então nenhum orçamento ou pagamento é alterado para aprovado
    E o evento não é publicado para autorizar execução
    E a ocorrência é registrada para investigação conforme política de segurança

  Cenário: Não duplicar estorno após reentrega da compensação
    Dado que Billing já confirmou o estorno de uma tentativa aprovada
    Quando a mesma solicitação de compensação é recebida novamente
    Então Billing retorna o resultado já persistido
    E não solicita um segundo estorno ao Mercado Pago

  Cenário: Estornar quando uma etapa posterior exigir compensação
    Dado que o pagamento foi aprovado
    E a Saga solicitou compensação por falha posterior
    Quando o estorno é solicitado ao provedor
    Então Billing registra o resultado do estorno
    E publica o evento de compensação apenas após confirmação
```

## Passos

1. Conferir a documentação oficial do Mercado Pago e decidir produto/API compatível com o fluxo de orçamento.
2. Criar adapter de provedor atrás de porta interna e configuração por ambiente.
3. Implementar criação de pagamento e persistência da referência/idempotência.
4. Implementar webhook com verificação de autenticidade, consulta autoritativa e proteção contra replay.
5. Implementar cancelamento/estorno e tratar estados que o provedor não permita reverter.
6. Validar no sandbox; registrar requisições/respostas anonimizadas e métricas de integração.

## Evidências

- Link para ADR/RFC e documentação oficial usada.
- Testes automatizados para assinatura, idempotência, retries e transições.
- Evidência de sandbox com tokens/IDs sensíveis removidos.

Registradas em 2026-10-13, PR #2 de `tech-challenge-billing`:

| Evidência | Onde | Resultado |
|---|---|---|
| Decisão e documentação oficial | [ADR-019](../../adr/ADR-019-integracao-mercado-pago-fase4.md), seção Referências | Pix pela API de Pagamentos, webhook assinado, reconciliação |
| Testes de domínio, casos de uso e contrato | `tests/OficinaMecanica.Billing.UnitTests` | 137 aprovados (77 novos). Estados que só avançam, divergência, falha técnica até ação manual, estorno, idempotência e os 9 canais de pagamento contra a spec |
| Testes de persistência, adapters, API e mensageria (PostgreSQL e RabbitMQ via Testcontainers) | `tests/OficinaMecanica.Billing.IntegrationTests` | 83 aprovados (56 novos). Comandos e reconciliações simultâneos, mesma notificação em várias réplicas, `401` para assinatura ausente/adulterada/antiga, cliente HTTP com retry e respostas gravadas, fluxo cobrança → aprovação → estorno pelo broker |
| Assinatura do webhook | `ValidadorAssinaturaTests` | HMAC igual ao calculado fora do código; recurso, requisição, segredo ou `ts` trocados são recusados |
| OpenAPI gerado | `docs/openapi/billing-v1.json` | 9 rotas: as 5 do orçamento, consulta do pagamento, webhook e simulador |
| Sandbox Mercado Pago | — | Pendente: credenciais e URL pública |