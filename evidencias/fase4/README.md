# Evidências — Fase 4

Evidências de execução real dos cards da Fase 4, para os critérios que pedem mais que testes automatizados. Cada arquivo diz de qual card é. A consolidação para a entrega fica no [CARD-43](../../cards/06-fase4-microsservicos-saga/CARD-43-documentacao-e-entrega.md).

| Arquivo | Card | Conteúdo |
|---|---|---|
| [card-37b-trace-os.json](card-37b-trace-os.json) | [CARD-37b](../../cards/06-fase4-microsservicos-saga/CARD-37b-api-e-contratos-os.md) | Trace do OS exportado do Jaeger: abertura da OS por HTTP e consumo de `PaymentApproved.v1` no mesmo trace e correlation ID |

## CARD-37b — trace de exemplo do OS

Gerado em 2026-10-08 com o OS de `main` (após o PR #2 de `tech-challenge-os`), PostgreSQL 16, RabbitMQ 3.13 e Jaeger 1.62 em containers locais.

O cenário usa um único `traceparent` (trace `972b6139bbd2f43cda1c89b948da857a`) e um único correlation ID (`6efad4b2-5f1b-409f-933d-37bceb82f987`):

1. `POST /os/ordens-servico` com `traceparent` e `X-Correlation-Id`. Abre a OS-2026-0003 e responde `201` com o mesmo `X-Correlation-Id`.
2. Uma `PaymentApproved.v1` publicada na exchange `saga-os.payment-approved.v1`, como o Billing publicaria, com `traceparent` no cabeçalho AMQP e o mesmo `correlationId` no envelope.
3. A mesma mensagem de novo (mesmo `messageId`).
4. Uma `PaymentApproved.v1` sem `amount`.

| Span | Duração | Resultado |
|---|---|---|
| `POST os/ordens-servico` | 9,0 ms | `201`, tag `correlation_id` |
| `saga-os.payment-approved.v1 process` (válida) | 22,1 ms | Gravada na inbox, tag `messaging.message.id` |
| `saga-os.payment-approved.v1 process` (duplicata) | 1,9 ms | Confirmada sem novo registro |
| `saga-os.payment-approved.v1 process` (sem `amount`) | 7,1 ms | `ERROR`; DLQ com `x-motivo` |

Os spans `oficina_os` são as consultas do EF Core ao PostgreSQL.

Conferido fora do trace:

- `InboxMensagens` tem uma linha só, a da mensagem válida.
- A DLQ `os.saga-os.payment-approved.v1.dlq` tem a mensagem inválida com `x-motivo`, `x-canal-origem`, `x-message-id` e `x-correlation-id`.
- A OS continua em `EmDiagnostico`. O efeito de negócio do evento é do [CARD-40](../../cards/06-fase4-microsservicos-saga/CARD-40-saga-e-bdd-integrado.md).

Para reproduzir, suba o OS com `RabbitMq__Enabled=true` e `Jaeger__Endpoint` apontando para o coletor (ver README de `tech-challenge-os`), repita os passos acima e exporte por `GET /api/traces/<traceId>` do Jaeger. Os segredos são gerados na hora e não fazem parte do arquivo.
