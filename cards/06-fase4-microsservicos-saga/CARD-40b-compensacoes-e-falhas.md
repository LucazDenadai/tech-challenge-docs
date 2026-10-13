# CARD-40b — Saga: compensações, falhas e cancelamento

**Tipo:** Implementação / Integração distribuída
**Status:** Concluído — PRs mergeados em `tech-challenge-os` (#5), `tech-challenge-operacoes` (#3), `tech-challenge-billing` (#4) e AsyncAPI (#37) em 2026-10-13; evidência de CI pendente (CARD-41)
**Depende de:** CARD-40a
**Bloqueia:** CARD-40c
**Repositórios alvo:** `tech-challenge-os` (Saga, API), `tech-challenge-operacoes` (início sem reserva e cancelamento de execução) e `tech-challenge-docs` (AsyncAPI)
**Decisão arquitetural:** [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md) (timeouts e compensações), [ADR-018](../../adr/ADR-018-contratos-assincronos-asyncapi-fase4.md) e [ADR-020](../../adr/ADR-020-saga-servico-sem-peca-cancelamento-acao-manual.md) (serviço sem peça, cancelamento antes do reparo e resolução manual)

---

## Contexto

O [CARD-40a](CARD-40a-orquestrador-saga-os.md) entrega o caminho normal e as rejeições sem efeito a desfazer. Este card trata o que acontece depois que um efeito já foi confirmado: reserva feita, pagamento aprovado ou execução iniciada. Compensação é uma nova ação de negócio, não rollback (ADR-017).

## Ponto de partida (2026-10-13)

- **OS:** a Saga já tem os estados de desvio no enum, mas nenhuma transição para eles. `PaymentDeclined`, `PaymentOutcomeUnknown`, `ExecutionStartRejected`, `ExecutionFailed`, `InventoryReleased` e os eventos de estorno chegam e contam como fora do fluxo. Os DTOs de `InventoryReleaseRequested` e `PaymentRefundRequested` existem, sem uso.
- **Cancelamento manual:** encerra a Saga em `Cancelled`, sem compensar.
- **Diagnóstico só com serviços:** para em `ManualActionRequired`, sem saída.
- **Operações:** consome `InventoryReleaseRequested` e `ExecutionStartRequested` (exige reserva ativa da OS). Não tem cancelamento de execução.
- **Billing:** consome `PaymentRefundRequested` (estorno total, idempotente) e publica `PaymentRefunded`, `PaymentRefundPending` e `PaymentRefundFailed`.

## Decisões de implementação (2026-10-13)

As decisões de negócio e de contrato estão no [ADR-020](../../adr/ADR-020-saga-servico-sem-peca-cancelamento-acao-manual.md). Decisões de código:

- **Contrato (exceção do ADR-020):** `ExecutionStartRequested.reservationId` fica opcional. Canais novos: `saga-os.execution-cancel-requested.v1` (`reason`, `executionId`), `saga-os.execution-cancelled.v1` (`executionId`) e `saga-os.execution-cancel-rejected.v1` (`reason`, `executionId`). As cópias da spec no OS e em Operações mudam no mesmo ciclo.
- **Transições novas:**

  | Estado atual | Mensagem ou ação | Novo estado | Comandos publicados |
  |---|---|---|---|
  | `AwaitingApproval` | `QuoteApproved` sem peças | `CreatingPayment` | `PaymentCreationRequested` |
  | `CreatingPayment`, `AwaitingPayment` | `PaymentOutcomeUnknown` | `ReconcilingPayment` | — |
  | `ReconcilingPayment` | `PaymentApproved` | `StartingExecution` | `ExecutionStartRequested` |
  | `CreatingPayment`, `AwaitingPayment`, `ReconcilingPayment` | `PaymentDeclined` | `ReleasingReservation` (ou `Cancelled` sem reserva) | `InventoryReleaseRequested` |
  | `ReleasingReservation` | `InventoryReleased` | `Cancelled` | — |
  | `StartingExecution` | `ExecutionStartRejected` | `Compensating` | `PaymentRefundRequested` e `InventoryReleaseRequested` (se houver reserva) |
  | `InExecution` | `ExecutionFailed` | `Compensating` | os mesmos |
  | `CancellingExecution` | `ExecutionCancelled` | `Compensating` | os mesmos |
  | `CancellingExecution` | `ExecutionCancelRejected` | `InExecution` | — |
  | `Compensating` | `PaymentRefunded` e `InventoryReleased` (os dois) | `Compensated` | — |
  | `Compensating` | `PaymentRefundPending` | `Compensating` (estorno pendente registrado) | — |
  | `Compensating` | `PaymentRefundFailed` | `ManualActionRequired` | — |

  Em `Compensating`, a Saga guarda o que já foi confirmado (estorno, liberação) e só termina quando os dois existem. Sem reserva, basta o estorno. `PaymentApproved` em `ReconcilingPayment` segue a mesma regra de valor/moeda do CARD-40a.
- **Cancelamento manual pela API:**

  | Estado da Saga | Efeito | Resposta |
  |---|---|---|
  | `Diagnosing`, `RequestingQuote`, `AwaitingApproval` | `Cancelled` | `204` |
  | `ReservingInventory`, `CreatingPayment`, `AwaitingPayment`, `ReconcilingPayment`, `StartingExecution` | Marca cancelamento pedido. A resposta do participante em andamento decide: reserva confirmada vira liberação; pagamento aprovado vira estorno e liberação; pagamento recusado vira liberação; execução iniciada vira `ExecutionCancelRequested`. | `202` |
  | `InExecution` | `CancellingExecution` e `ExecutionCancelRequested` | `202` |
  | Desvio em andamento (`ReleasingReservation`, `Compensating`, `CancellingExecution`) | Nada novo: a Saga já está encerrando | `202` |
  | `ManualActionRequired` | Recusado: usar a resolução do Admin | `422` |
  | Terminal | Recusado | `422` |

  O status da OS muda pela Saga (`Cancelada` em `Cancelled` e `Compensated`), não mais pela rota. A rota de entrega não muda.
- **Timeouts:** estados que esperam resposta automática de um participante (`RequestingQuote`, `ReservingInventory`, `CreatingPayment`, `StartingExecution`, `ReleasingReservation`, `Compensating`, `CancellingExecution`) têm prazo `Saga:PrazoRespostaMinutos` (padrão 5). Ao vencer, um serviço em segundo plano reenvia os comandos pendentes da etapa com a mesma `idempotencyKey`, até `Saga:MaxReenvios` (padrão 3). Esgotado, a Saga vai para `ManualActionRequired`, com log de erro e a métrica `os_saga_acao_manual`. Sem prazo no OS: `Diagnosing` e `InExecution` (trabalho humano), `AwaitingApproval` (Billing expira), `AwaitingPayment` e `ReconcilingPayment` (Billing reconcilia e mantém a reserva, ADR-017).
- **Ação manual:** a Saga guarda o estado de retomada e a causa. `POST /os/ordens-servico/{id}/saga/resolucao` (`Admin`) com `{ "acao": "Cancelar" | "Retomar", "motivo": "..." }`. `Retomar` volta ao estado de retomada e reenvia os comandos da etapa; é recusado (`422`) para pagamento divergente. A resolução grava transição com o motivo e o usuário do token.
- **`idempotencyKey` das compensações:** `os:<osId>:release`, `os:<osId>:refund` e `os:<osId>:cancel-execution`.
- **Operações:** `ExecutionStartRequested` sem `reservationId` é aceito só se o diagnóstico não tem peças; conclusão e falha sem reserva não movem estoque. `ExecutionCancelRequested` cancela execução `Diagnosticada` ou `NaFila` (novo estado `Cancelada` na execução) e recusa as demais, com motivo. Repetição com a mesma chave republica o resultado gravado.
- **Fora deste card:** multa e estorno parcial (evolução no ADR-020); lease da reserva (evolução no ADR-017).

## Escopo

- Pagamento recusado, expirado ou desconhecido, com liberação ou reconciliação.
- Compensação depois do pagamento: execução recusada, falha e cancelamento antes do reparo.
- Estorno pendente ou falho como estado explícito.
- Serviço sem peça de ponta a ponta (ADR-020).
- Timeouts com reenvio e ação manual.
- Cancelamento manual pela Saga e resolução de ação manual pelo Admin.
- Mudanças de contrato no AsyncAPI e em Operações.

## Implementação (2026-10-13)

- **Repositórios:** `tech-challenge-docs` PR #37 (AsyncAPI), `tech-challenge-operacoes` PR #3 (3 commits), `tech-challenge-os` PR #5 (7 commits: domínio, contratos, orquestrador, casos de uso e API, correções, testes pelo broker, README/OpenAPI) e `tech-challenge-billing` PR #4 (só a cópia da spec).
- **Comandos por etapa:** o domínio informa os comandos de cada estado (`ComandosDaEtapa`). A transição por mensagem, o reenvio por prazo, a retomada pelo Admin e o cancelamento manual usam a mesma regra, e por isso geram a mesma `idempotencyKey`.
- **Efeito depois do encerramento (decisão de implementação):** `InventoryReserved` ou `PaymentApproved` que chega com a Saga já cancelada gera `InventoryReleaseRequested` ou `PaymentRefundRequested`. Sem isso, um prazo esgotado seguido de cancelamento podia deixar reserva presa ou pagamento sem estorno.
- **`CompensationPending`:** fica no enum, sem uso. Estorno pendente fica em `Compensating` com a marca de pendência; liberação não confirmada cai no prazo e, esgotado, em ação manual.
- **Retomada depois de estorno recusado:** reenvia o pedido, mas o Billing republica o resultado gravado (`PaymentRefundFailed`). Na prática, a resolução é `Cancelar` depois de tratar o estorno fora do sistema.
- **Correções no caminho:** no C# 14, `array.Contains` dentro de expressão LINQ vira a sobrecarga de `ReadOnlySpan`, que o EF não traduz; a consulta de prazos usa `Enumerable.Contains`. Atributos de validação de um record ficam no parâmetro do construtor, não em `[property:]`.

### Pendências

- **Multa e estorno parcial:** evolução no ADR-020.
- **E-mail de mudança de status:** as transições pela Saga não enviam o e-mail que a rota manual da Fase 3 enviava. Fica para o CARD-42/43, se o vídeo pedir.
- **Vários réplicas do serviço de prazos:** duas réplicas podem reenviar o mesmo comando ao mesmo tempo; é seguro pela `idempotencyKey` e pelo `xmin`. A trava da outbox continua no CARD-41.

## Critérios de aceite

- [x] Cada linha da tabela de compensações do ADR-017 tem caminho implementado e teste de integração; itens marcados como evolução no Escopo do MVP não são implementados.
- [x] Pagamento aprovado não é marcado como desfeito até a confirmação do estorno; estorno pendente fica explícito.
- [x] Resultado de pagamento desconhecido mantém a Saga em reconciliação, sem recusar, cobrar de novo ou liberar a reserva.
- [x] Falha após aprovação do pagamento aciona liberação da reserva não consumida e estorno total.
- [x] Toda etapa tem timeout e retry explícitos; ao esgotar, a Saga vai para estado de ação manual com alerta. *(Etapas que esperam participante. Diagnóstico e reparo são trabalho humano; aprovação e pagamento têm prazo no Billing.)*
- [x] Cancelamento manual passa pela Saga e não deixa reserva presa nem pagamento sem estorno.
- [x] Decisão sobre cancelamento de execução registrada (ADR ou card) e refletida no AsyncAPI, se houver comando novo. *(Decisão: [ADR-020](../../adr/ADR-020-saga-servico-sem-peca-cancelamento-acao-manual.md).)*
- [x] OS só com serviços vai da abertura à conclusão sem reserva.
- [x] `ManualActionRequired` tem saída pelo Admin, com motivo no histórico.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Compensar falhas da Saga

  Cenário: Liberar reserva quando o pagamento é recusado
    Dado uma Saga em AwaitingPayment com estoque reservado
    Quando chega PaymentDeclined
    Então OS pede a liberação da reserva
    E a Saga só termina em Cancelled depois de InventoryReleased

  Cenário: Compensar falha de execução após pagamento aprovado
    Dado que o pagamento foi aprovado
    Quando chega ExecutionFailed
    Então OS pede estorno total ao Billing e liberação da parte não consumida a Operações
    E a Saga só termina em Compensated depois das duas confirmações

  Cenário: Manter a reserva com pagamento desconhecido
    Dado uma Saga em AwaitingPayment
    Quando chega PaymentOutcomeUnknown
    Então a Saga fica em ReconcilingPayment
    E nenhum comando de liberação ou nova cobrança é publicado

  Cenário: Cancelar execução ainda na fila
    Dado uma Saga em InExecution com a execução na fila em Operações
    Quando o atendente cancela a OS
    Então a API responde 202 e OS pede o cancelamento a Operações
    E depois de ExecutionCancelled a Saga pede estorno total e liberação da reserva
    E termina em Compensated com a OS Cancelada

  Cenário: Recusar cancelamento com o reparo em andamento
    Dado uma Saga em CancellingExecution
    Quando chega ExecutionCancelRejected
    Então a Saga volta a InExecution
    E nenhum estorno é pedido

  Cenário: Concluir OS só com serviço
    Dado um diagnóstico só com serviços
    Quando o orçamento é aprovado e o pagamento confirmado
    Então nenhuma reserva é pedida
    E a execução começa sem reservationId e a Saga termina em Completed

  Cenário: Admin resolve ação manual
    Dado uma Saga em ManualActionRequired por prazo esgotado
    Quando o Admin escolhe Retomar com um motivo
    Então a Saga volta à etapa e reenvia o comando com a mesma idempotencyKey
    E o motivo fica no histórico
```

## Passos

1. Contrato: spec no `tech-challenge-docs` e cópias nos repositórios.
2. Operações: início sem reserva e cancelamento de execução.
3. OS, domínio: transições de desvio, compensação com confirmações e estado de retomada.
4. OS, orquestrador: comandos de compensação e eventos novos.
5. OS: cancelamento manual pela Saga e resolução de ação manual.
6. OS: timeouts com reenvio.
7. Testes de integração de cada fronteira de falha, README e OpenAPI.

## Evidências

- Testes de integração por linha da tabela de compensações do ADR-017.
- Trace correlacionado de uma compensação completa.

Registradas em 2026-10-13:

| Evidência | Onde | Resultado |
|---|---|---|
| Domínio das compensações | `tech-challenge-os`, `SagaCompensacaoTests` | Pagamento recusado e desconhecido, falha e cancelamento com duas confirmações, estorno pendente e falho, cancelamento por estado, efeito atrasado, prazo, retomada e cancelamento pelo Admin |
| Fluxos pelo broker (PostgreSQL e RabbitMQ via Testcontainers) | `tech-challenge-os`, `SagaCompensacaoFluxoTests` | Recusa → liberação → `Cancelled`; desconhecido sem comando; falha → `Compensated`; cancelamento na fila (`202`) → `Compensated`; recusa em reparo; serviço sem peça até `Completed`; prazo com reenvio da mesma chave e retomada pelo Admin; pagamento divergente só cancelável |
| Operações | `tech-challenge-operacoes`, `ExecucaoUseCasesTests` e `MensageriaSagaTests` | Início sem reserva só sem peças; cancelamento na fila, recusa em reparo, repetição com a mesma chave, gravação no DynamoDB Local |
| Totais | | OS: 249 unitários e 50 de integração; Operações: 141 e 53; Billing: 137 unitários com a spec nova |
