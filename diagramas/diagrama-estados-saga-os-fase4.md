# Estados — Saga da OS na Fase 4

**Proposta para avaliação:** estados para a opção de orquestração descrita no ADR-017. A máquina será congelada após confirmação do CARD-36.

```mermaid
stateDiagram-v2
    [*] --> RequestingQuote
    RequestingQuote --> AwaitingApproval: QuoteReady
    RequestingQuote --> Cancelled: QuoteRejected / timeout de orçamento
    AwaitingApproval --> ReservingInventory: QuoteApproved
    AwaitingApproval --> Cancelled: QuoteDeclined / QuoteExpired
    ReservingInventory --> CreatingPayment: InventoryReserved
    ReservingInventory --> Cancelled: InventoryReservationRejected
    CreatingPayment --> AwaitingPayment: PaymentPending
    CreatingPayment --> ReconcilingPayment: Timeout / resultado desconhecido
    AwaitingPayment --> StartingExecution: PaymentApproved
    AwaitingPayment --> ReleasingReservation: PaymentDeclined / cancelado
    AwaitingPayment --> ReconcilingPayment: Webhook ausente / timeout técnico
    ReconcilingPayment --> StartingExecution: Provedor confirma aprovado
    ReconcilingPayment --> ReleasingReservation: Provedor confirma terminal não aprovado
    ReconcilingPayment --> ManualActionRequired: Limite de reconciliação excedido
    StartingExecution --> InExecution: ExecutionStarted
    StartingExecution --> Compensating: ExecutionStartRejected após pagamento aprovado
    InExecution --> Completed: ExecutionCompleted
    InExecution --> Compensating: ExecutionFailed
    ReleasingReservation --> Cancelled: InventoryReleased
    ReleasingReservation --> ManualActionRequired: Liberação não confirmada
    Compensating --> CompensationPending: Solicitar estorno/liberação
    CompensationPending --> Compensated: Estorno e liberação confirmados
    CompensationPending --> ManualActionRequired: Falha/resultado incerto após retries
    ManualActionRequired --> ReconcilingPayment: Replay auditado
    ManualActionRequired --> CompensationPending: Replay auditado
    Completed --> [*]
    Cancelled --> [*]
    Compensated --> [*]
```

`ManualActionRequired` só sai por ação auditada ou reconciliação que determine um resultado seguro; replay não pode duplicar efeitos já confirmados.