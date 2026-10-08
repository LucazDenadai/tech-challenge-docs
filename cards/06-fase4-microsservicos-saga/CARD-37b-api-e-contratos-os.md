# CARD-37b — API, contratos e histórico da OS

**Tipo:** Implementação / API
**Status:** Implementado — branch `feat/card-37b-api-contratos-os` em `tech-challenge-os`, testes verificados localmente em 2026-10-07; aguardando PR, CI (CARD-41), coleção Postman e trace de exemplo
**Depende de:** CARD-36, CARD-37a
**Bloqueia:** CARD-40, CARD-43
**Repositório alvo:** `tech-challenge-os`
**Decisão arquitetural:** Contratos aprovados no CARD-36 ([ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md), [ADR-018](../../adr/ADR-018-contratos-assincronos-asyncapi-fase4.md))

---

## Contexto

OS é a autoridade sobre a ordem e precisa expor operações síncronas necessárias aos clientes e receber eventos dos serviços parceiros. O contrato deve preservar a propriedade do estado e permitir correlação de chamadas/mensagens sem acoplar a API ao esquema de dados de outro serviço.

## Decisões de implementação (2026-10-06)

- **API:** o projeto executável `OficinaMecanica.OS.API` expõe toda a API sob ownership do OS, não só a OS: auth/JWT dos funcionários, clientes, veículos, usuários, abertura/listagem/detalhe da OS, status, histórico, transições manuais (`Entregue`, cancelamento), endpoint interno de busca de cliente por CPF para a Lambda (ADR-015), health/readiness e Swagger. O `Program.cs` chama o seed de demonstração com `Seed:SenhaUsuarios`.
- **Eventos (fronteira com o CARD-40):** o 37b entrega só a infraestrutura de consumo dos canais que o OS recebe no AsyncAPI: conexão autenticada ao RabbitMQ, declaração das filas `os.<canal>` e das DLQs `os.<canal>.dlq`, validação do envelope e do payload, deduplicação por `messageId` em uma tabela inbox no PostgreSQL do OS, envio para a DLQ com motivo e propagação do `correlationId` em logs e traces. O efeito de negócio (transição da Saga e do status da OS), a outbox e a publicação de comandos ficam no CARD-40, porque o ADR-017 não permite escrever o status fora da transição da Saga.

## Implementação (2026-10-07)

- **Rotas:** tudo sob `/os/*`. O `PUT /status` da Fase 3 sai. A API só expõe as transições manuais `POST /os/ordens-servico/{id}/entrega` e `/cancelamento`; o resto do status muda pela Saga (ADR-017). Há `GET /os/filiais` só de leitura, para obter o `filialId` da abertura.
- **Endpoint da Lambda:** `POST /os/interno/clientes/busca-por-cpf` com o CPF no corpo, autenticado pelo cabeçalho `X-Api-Key` (`Interno:ApiKey`). Devolve só `clienteId` e `ativo`. O JWT não substitui a chave, e a rota não deve ser publicada no API Gateway.
- **Autorização:** cada controller declara as roles. Token de cliente (Lambda) não acessa cadastros. O acompanhamento por número só devolve OS do `cliente_id` do token. Login devolve `401` igual para e-mail inexistente, senha errada e usuário desativado, que antes conseguia logar.
- **Contratos:** os 21 eventos que o OS consome viram records C#. Bases como `RefundResult`, `PaymentRef`, `QuoteRef`, `QuoteDecision` e `Rejection` espelham os schemas comuns da spec. Cada `messageType` mantém seu record e canal. A spec é copiada em `contratos/` no repositório do OS. Os testes de contrato comparam catálogo, exemplos e mutações inválidas com o JSON Schema dessa cópia.
- **Consumidor:** usa `RabbitMQ.Client` direto, não MassTransit, porque o MassTransit impõe envelope e topologia próprios, diferentes dos nomes do ADR-018. A inbox é a tabela `InboxMensagens`: `messageId` como PK, gravação por `INSERT ... ON CONFLICT DO NOTHING` e payload em `jsonb` para o CARD-40 processar. Falha transitória tem 3 retentativas (1 s, 5 s, 10 s) antes da DLQ (ADR-017). A DLQ recebe o corpo original com os cabeçalhos `x-motivo`, `x-canal-origem`, `x-message-id` e `x-correlation-id`.
- **Spec:** os testes de contrato acharam duas descrições sem aspas no `asyncapi-saga-os.yaml` (`correlationId` e `providerReference`). A vírgula truncava o texto e criava uma chave inválida no schema. Corrigido nos dois repositórios, sem mudar campos nem versão.

### Pendências

- **Lambda:** ainda consulta o RDS. Nenhum card cobre a troca dela para o endpoint interno (repositório `tech-challenge-lambda`). Decidir se entra no CARD-41 ou em card próprio.
- **Cancelamento manual:** cancelar a partir de `AguardandoPagamento` ou `EmExecucao` só muda o status; não dispara liberação de reserva nem estorno. O CARD-40 precisa levar o cancelamento manual pela Saga, com compensação.
- **Cópia da spec:** a cópia em `tech-challenge-os/contratos/` é sincronizada à mão. Uma checagem de divergência no CI cabe no CARD-41.
- **Evidências:** falta a coleção Postman e o trace de exemplo. O trace exige subir OS, RabbitMQ e o coletor.

## Critérios de aceite

- [x] Endpoints cobrem cadastros, auth, abertura, consulta de status e consulta de histórico com validação, autorização e códigos HTTP documentados.
- [x] Endpoint interno de busca por CPF é autenticado e devolve só o necessário para a Lambda.
- [x] API tem Swagger/OpenAPI atualizado e exemplos sem credenciais ou dados pessoais reais.
- [x] Eventos recebidos seguem o schema aprovado, são autenticados conforme infraestrutura e têm correlation ID.
- [x] Eventos duplicados ou incompatíveis não geram mutação silenciosa: duplicata é confirmada sem novo registro; tipo/versão desconhecido ou payload inválido vai para a DLQ com motivo e log.
- [x] Mudanças de contrato seguem estratégia de versionamento/compatibilidade aprovada.
- [x] Testes de API cobrem happy path, validação, autorização e cenários de falha.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Consultar a OS sem expor bancos internos

  Cenário: Consultar status e histórico
    Dado que existe uma OS no banco do serviço OS
    Quando um consumidor autorizado consulta status e histórico pela API
    Então recebe a situação atual e as transições permitidas para exibição
    E a resposta não expõe identificadores internos de banco ou credenciais

  Cenário: Solicitação inválida de abertura
    Dado que a requisição não contém os campos obrigatórios do contrato
    Quando o cliente solicita a abertura da OS
    Então a API retorna erro de validação documentado
    E nenhuma OS é persistida

  Cenário: Reentregar um evento já recebido
    Dado que o OS já registrou um evento com determinado messageId
    Quando a mesma mensagem é entregue novamente
    Então a mensagem é confirmada sem criar novo registro

  Cenário: Receber evento incompatível
    Dado uma mensagem com tipo, versão ou payload fora do contrato
    Quando o OS a consome
    Então a mensagem vai para a DLQ do canal com o motivo
    E nenhum estado do OS é alterado
```

## Passos

1. Criar o projeto API com controllers, DTOs, autorização, validação, health e Swagger.
2. Implementar o consumidor dos canais de Billing e Operações definidos no CARD-36: envelope, inbox, DLQ e correlation ID (sem efeito de negócio, que fica no CARD-40).
3. Adicionar logs estruturados e trace context.
4. Publicar Swagger e casos de exemplo para Postman da solução.

## Evidências

- OpenAPI gerado, testes de API e coleção Postman atualizada.
- Trace de exemplo com correlation ID propagado.

Registradas em 2026-10-07, branch `feat/card-37b-api-contratos-os`:

| Evidência | Onde | Resultado |
|---|---|---|
| OpenAPI gerado | `docs/openapi/os-v1.json` | 18 rotas sob `/os/*` |
| Testes unitários e de contrato | `tests/OficinaMecanica.OS.UnitTests` | 150 aprovados |
| Testes de API, migrations e mensageria (PostgreSQL e RabbitMQ via Testcontainers) | `tests/OficinaMecanica.OS.IntegrationTests` | 33 aprovados |
| Coleção Postman | — | Pendente |
| Trace de exemplo | — | Pendente |
