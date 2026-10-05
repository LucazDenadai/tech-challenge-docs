# Sequência — Saga da OS na Fase 4

**Estratégia aprovada:** orquestração pelo módulo Saga do serviço OS (ADR-017, aceito).

```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant OS as Serviço OS / Orchestrator
    participant OSDB as PostgreSQL OS
    participant Broker as Broker
    participant Billing as Serviço Billing
    participant MP as Mercado Pago
    participant Ops as Serviço Operações
    participant OpsSQL as PostgreSQL Operações
    participant OpsNoSQL as DynamoDB Execução

    Cliente->>OS: Abrir OS (filialId, veículo)
    OS->>OSDB: Salvar OS + Saga(Diagnosing) + outbox
    OS-->>Cliente: OS criada; diagnóstico em andamento
    OS->>Broker: DiagnosisRequested.v1 (correlationId, osId, filialId)
    Broker->>Ops: DiagnosisRequested.v1
    Ops->>OpsNoSQL: Criar registro de execução (status=Diagnosing)
    Ops->>OpsNoSQL: Técnico registra diagnóstico (status=Diagnosed, fora da fila) + outbox transacional
    Ops->>Broker: DiagnosisCompleted.v1 (itens + snapshot de preços)
    Broker->>OS: DiagnosisCompleted.v1
    OS->>OSDB: Saga = RequestingQuote + outbox
    OS->>Broker: QuoteRequested.v1 (correlationId, osId, filialId, itens + preços)
    Broker->>Billing: QuoteRequested.v1
    Billing->>Billing: Validar pedido e criar orçamento
    Billing->>Broker: QuoteReady.v1 (quoteId, version, amount, currency, expiresAt)
    Broker->>OS: QuoteReady.v1
    OS->>OSDB: Saga = AwaitingApproval; registrar referência/versão
    Cliente->>Billing: REST: aprovar versão do orçamento (quoteId, version)
    Billing->>Billing: Validar versão/validade e registrar decisão idempotente + outbox
    Billing-->>Cliente: 200 aprovado / 409 versão desatualizada / 410 expirado
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
        Ops->>OpsNoSQL: Mover execução diagnosticada para fila (status=Queued/Started) + outbox transacional
        Ops->>Broker: ExecutionStarted.v1
        Broker->>OS: ExecutionStarted.v1
        Ops->>OpsNoSQL: Atualizar progresso do reparo
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
        OS->>OSDB: Saga=ReconcilingPayment; manter reserva
        Note over OS,Billing: Job de Billing reconcilia com a mesma idempotencyKey; timeout não é recusa
    end

    opt Falha após pagamento aprovado (início rejeitado ou execução falhou)
        Ops-->>OS: ExecutionStartRejected.v1 ou ExecutionFailed.v1 (itens consumidos)
        OS->>Broker: InventoryReleaseRequested.v1 + PaymentRefundRequested.v1 (estorno total)
        Broker->>Ops: InventoryReleaseRequested.v1
        Broker->>Billing: PaymentRefundRequested.v1
        Ops->>OpsSQL: Registrar baixa do consumido e liberar reserva não consumida
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