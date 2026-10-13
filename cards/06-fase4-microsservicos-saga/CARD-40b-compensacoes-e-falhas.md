# CARD-40b — Saga: compensações, falhas e cancelamento

**Tipo:** Implementação / Integração distribuída
**Status:** To Do — decisões de implementação a definir no início do card
**Depende de:** CARD-40a
**Bloqueia:** CARD-40c
**Repositórios alvo:** `tech-challenge-os`; `tech-challenge-operacoes` e `tech-challenge-docs` (AsyncAPI) se o cancelamento de execução ganhar comando novo
**Decisão arquitetural:** [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md) (tabelas de timeouts e compensações) e [ADR-018](../../adr/ADR-018-contratos-assincronos-asyncapi-fase4.md)

---

## Contexto

O [CARD-40a](CARD-40a-orquestrador-saga-os.md) entrega o caminho normal e as rejeições sem efeito a desfazer. Este card trata o que acontece depois que um efeito já foi confirmado: reserva feita, pagamento aprovado ou execução iniciada. Compensação é uma nova ação de negócio, não rollback (ADR-017).

## Escopo

- Pagamento recusado ou expirado (`PaymentDeclined`): liberar a reserva; `ReleasingReservation` → `Cancelled` após `InventoryReleased`, ou `CompensationPending`.
- Pagamento desconhecido (`PaymentOutcomeUnknown`): `ReconcilingPayment`, mantendo a reserva, até `PaymentApproved` ou `PaymentDeclined`.
- Execução rejeitada ou falha depois do pagamento (`ExecutionStartRejected`, `ExecutionFailed`): estorno total e liberação da parte não consumida como passos separados; `Compensating` → `Compensated` só com os dois confirmados, ou `ManualActionRequired`.
- Estorno pendente ou falho (`PaymentRefundPending`, `PaymentRefundFailed`): estado explícito, sem declarar a Saga compensada.
- Timeouts e retries da tabela do ADR-017, com alerta ao esgotar.
- Cancelamento manual da OS pela Saga (pendência do [CARD-37b](CARD-37b-api-e-contratos-os.md)): a partir de `AguardandoPagamento` ou `EmExecucao`, disparar liberação e estorno conforme o estado.
- Cancelamento de execução já na fila ou em reparo (pendência do [CARD-38b](CARD-38b-fila-execucao.md)): decidir se entra um comando novo no AsyncAPI ou se o cancelamento fica bloqueado a partir de `EmExecucao`.
- Diagnóstico só com serviços (pendência do [CARD-40a](CARD-40a-orquestrador-saga-os.md)): hoje para em `ManualActionRequired`, porque a reserva exige ao menos uma peça e o início da execução exige reserva. Decidir a mudança de contrato.
- Saída de `ManualActionRequired`: hoje não há transição de saída; definir a recuperação auditável.

## Critérios de aceite

- [ ] Cada linha da tabela de compensações do ADR-017 tem caminho implementado e teste de integração; itens marcados como evolução no Escopo do MVP não são implementados.
- [ ] Pagamento aprovado não é marcado como desfeito até a confirmação do estorno; estorno pendente fica explícito.
- [ ] Resultado de pagamento desconhecido mantém a Saga em reconciliação, sem recusar, cobrar de novo ou liberar a reserva.
- [ ] Falha após aprovação do pagamento aciona liberação da reserva não consumida e estorno total.
- [ ] Toda etapa tem timeout e retry explícitos; ao esgotar, a Saga vai para estado de ação manual com alerta.
- [ ] Cancelamento manual passa pela Saga e não deixa reserva presa nem pagamento sem estorno.
- [ ] Decisão sobre cancelamento de execução registrada (ADR ou card) e refletida no AsyncAPI, se houver comando novo.

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
```

## Passos

1. Definir as decisões de implementação (cancelamento de execução, timeouts, alertas) e registrar neste card.
2. Implementar as transições de desvio e os comandos de compensação.
3. Implementar timeouts e o estado de ação manual.
4. Levar o cancelamento manual pela Saga.
5. Testes de integração de cada fronteira de falha.

## Evidências

- Testes de integração por linha da tabela de compensações do ADR-017.
- Trace correlacionado de uma compensação completa.
