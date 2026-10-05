# Sequência — Saga da OS na Fase 4

**Estratégia em avaliação:** orquestração pelo módulo Saga do serviço OS (ADR-017). O fluxo é uma proposta para comparação; não implementar antes da confirmação do CARD-36.

```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant OS as Serviço OS / Orchestrator proposto
    participant OSDB as PostgreSQL OS
    participant Broker as Broker
    participant Billing as Serviço Billing
    participant MP as Mercado Pago
    participant Ops as Serviço Operações
    participant OpsSQL as PostgreSQL Operações
    participant OpsNoSQL as DynamoDB Execução

    Cliente->>OS: Abrir OS (filialId, itens)
    OS->>OSDB: Salvar OS + Saga(RequestingQuote) + outbox
    OS-->>Cliente: OS criada; orçamento em preparação
    OS->>Broker: QuoteRequested.v1 (correlationId, osId, filialId)
    Broker->>Billing: QuoteRequested.v1
    Billing->>Billing: Validar pedido e criar orçamento
    Billing->>Broker: QuoteReady.v1 (quoteId, version, amount, currency, expiresAt)
    Broker->>OS: QuoteReady.v1
    OS->>OSDB: Saga = AwaitingApproval; registrar referência/versão
    Cliente->>OS: Aprovar versão do orçamento
    OS->>Broker: QuoteApprovalRequested.v1 (quoteId, version)
    Broker->>Billing: QuoteApprovalRequested.v1
    Billing->>Billing: Registrar decisão idempotente
    Billing->>Broker: QuoteApproved.v1
    Broker->>OS: QuoteApproved.v1
    OS->>OSDB: Saga = ReservingInventory + outbox
    OS->>Broker: InventoryReservationRequested.v1 (filialId, items, idempotencyKey)
    Broker->>Ops: InventoryReservationRequested.v1
    Ops->>OpsSQL: Reservar por (filialId, pecaId) + outbox
    Ops->>Broker: InventoryReserved.v1
    Broker->>OS: InventoryReserved.v1
    OS->>OSDB: Saga = CreatingPayment + outbox
    OS->>Broker: PaymentCreationRequested.v1 (quoteId, amount, currency, idempotencyKey)
    Broker->>Billing: PaymentCreationRequested.v1
    Billing->>MP: Criar tentativa de pagamento
    MP-->>Billing: Referência/estado inicial
    Billing->>Billing: Persistir PaymentPending + outbox
    Billing->>Broker: PaymentPending.v1
    Broker->>OS: PaymentPending.v1
    OS->>OSDB: Saga = AwaitingPayment
    Cliente->>MP: Concluir pagamento
    MP->>Billing: Webhook
    Billing->>MP: Consultar/reconciliar recurso e validar assinatura
    MP-->>Billing: Estado autoritativo
    alt Pagamento aprovado
        Billing->>Broker: PaymentApproved.v1
        Broker->>OS: PaymentApproved.v1
        OS->>OSDB: Saga = StartingExecution + outbox
        OS->>Broker: ExecutionStartRequested.v1
        Broker->>Ops: ExecutionStartRequested.v1
        Ops->>OpsNoSQL: Criar execução (status=Queued/Started) + outbox transacional
        Ops->>Broker: ExecutionStarted.v1
        Broker->>OS: ExecutionStarted.v1
        Ops->>OpsNoSQL: Atualizar diagnóstico/progresso
        Ops->>OpsSQL: Confirmar consumo real ao concluir; outbox
        Ops->>Broker: ExecutionCompleted.v1
        Broker->>OS: ExecutionCompleted.v1
        OS->>OSDB: OS=Concluída; Saga=Completed + histórico
        OS-->>Cliente: Status final consultável
    else Pagamento recusado
        Billing->>Broker: PaymentDeclined.v1
        Broker->>OS: PaymentDeclined.v1
        OS->>Broker: InventoryReleaseRequested.v1
        Broker->>Ops: InventoryReleaseRequested.v1
        Ops->>OpsSQL: Liberar reserva idempotentemente
        Ops->>Broker: InventoryReleased.v1
        Broker->>OS: InventoryReleased.v1
        OS->>OSDB: Saga=Cancelled; OS sem cobrança
    else Resultado técnico desconhecido
        Billing->>Broker: PaymentOutcomeUnknown.v1
        Broker->>OS: PaymentOutcomeUnknown.v1
        OS->>OSDB: Saga=ReconcilingPayment; manter reserva dentro do TTL
        Note over OS,Billing: Reconciliar com a mesma idempotencyKey; renovar lease da reserva; timeout não é recusa
    end

    opt Falha após pagamento aprovado antes de executar trabalho
        Ops-->>OS: ExecutionStartRejected.v1 ou falha definitiva
        OS->>Broker: InventoryReleaseRequested.v1 + PaymentRefundRequested.v1
        Broker->>Ops: InventoryReleaseRequested.v1
        Broker->>Billing: PaymentRefundRequested.v1
        Ops->>OpsSQL: Liberar reserva não consumida
        Billing->>MP: Solicitar estorno
        MP-->>Billing: Estado do estorno
        Ops-->>OS: InventoryReleased.v1
        Billing-->>OS: PaymentRefunded.v1 ou RefundPending.v1
        OS->>OSDB: Compensated somente após ambas as confirmações
    end
```

## Regras do fluxo

- Todos os contratos incluem `messageId`, `messageType`, `schemaVersion`, `producer`, `occurredAtUtc`, `correlationId`, `causationId`, `osId`, `filialId` e `idempotencyKey` quando há efeito mutável.
- A outbox local é gravada no mesmo commit do store dono do efeito; operações entre stores/serviços são coordenadas pela Saga, nunca por transação distribuída.
- OS é o único writer do status geral da OS e da Saga. Billing é o único writer do orçamento/pagamento. Operações é o único writer do estoque e da execução.
- Estados `PaymentOutcomeUnknown`, `RefundPending` e `ManualActionRequired` são não terminais; não significam recusa, estorno concluído ou sucesso.
- A reserva usa a filial da OS. Estoque de outra filial não é consumido implicitamente.