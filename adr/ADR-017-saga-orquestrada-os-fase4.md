# ADR-017 — Avaliação de estratégia Saga para OS

**Status:** Proposto — recomendação para avaliação; não aprovado para implementação
**Data:** 2026-10-05
**Autores:** Time Tech Challenge — Fase 4
**Relaciona-se a:** [ADR-014](ADR-014-limites-microsservicos-fase4.md), [ADR-015](ADR-015-ownership-e-infraestrutura-fase4.md), [ADR-016](ADR-016-bancos-sql-nosql-fase4.md), [CARD-36](../cards/06-fase4-microsservicos-saga/CARD-36-contratos-e-desenho-saga.md)

---

## Contexto

O fluxo da Fase 4 atravessa os serviços OS, Billing e Operações. Cada serviço é dono do próprio banco e não pode consultar tabelas de outro serviço. A Saga precisa acompanhar o progresso, tolerar duplicidade/restart, compensar efeitos já confirmados e demonstrar o tratamento de falha.

O estado geral da OS pertence a OS, tornando-o o local mais direto para hospedar a coordenação sem criar um quarto microsserviço. Billing continua dono do orçamento/pagamento e Operações do estoque/execução. O orchestrator nunca grava diretamente nas bases desses participantes.

---

## Recomendação para avaliação

Recomenda-se **Saga orquestrada pelo serviço OS**. A coordenação seria um módulo interno de OS, persistido na instância PostgreSQL exclusiva de OS junto a uma outbox local. Não criaria um quarto serviço e não mudaria os limites do ADR-014. Esta alternativa não está aprovada: confirmar após ler as referências e validar o desenho com os responsáveis pelos três serviços.

### Ordem do fluxo normal

1. OS cria a ordem e a Saga no mesmo commit local, inicialmente `RequestingQuote`, e publica `QuoteRequested` via outbox.
2. Billing valida o pedido, cria uma versão de orçamento sob seu ownership e publica `QuoteReady` ou `QuoteRejected`.
3. Cliente autorizado aprova ou rejeita uma versão específica do orçamento. Billing é autoridade da aprovação; uma resposta para versão antiga é recusada.
4. Após aprovação, OS pede a Operações uma reserva de peças por `(filialId, pecaId)`. Se não houver saldo na filial, a Saga é cancelada sem criar pagamento.
5. Após `InventoryReserved`, OS pede a Billing a criação idempotente de uma tentativa de pagamento para a versão/valor/moeda aprovados. Billing cria a tentativa no Mercado Pago e publica `PaymentPending` com a referência do provedor.
6. O webhook é recebido e validado por Billing. Só depois de reconciliar com o provedor Billing publica `PaymentApproved`, `PaymentDeclined` ou o estado técnico de resultado desconhecido.
7. Após `PaymentApproved`, OS comanda Operações a iniciar a execução reservada. A execução registra diagnóstico/progresso no DynamoDB; o estoque permanece no PostgreSQL de Operações.
8. Operações conclui o trabalho, consome da reserva a quantidade efetivamente utilizada e publica `ExecutionCompleted`. OS registra o status/histórico final da ordem e encerra a Saga como `Completed`.

Comandos/eventos entre serviços usam o broker. REST é usado para APIs síncronas de entrada/consulta e chamadas em que o chamador precisa de resposta imediata; não se usa REST como coordenador oculto de todo o processo distribuído. Mercado Pago chama o webhook público de Billing, nunca o banco ou o orchestrator diretamente.

### Responsabilidades e stores

| Participante | Responsabilidade na Saga | Persistência própria usada |
|---|---|---|
| OS | Orchestrator, transições gerais da ordem, estado da Saga, outbox e histórico de OS | PostgreSQL OS |
| Billing | Versão/decisão do orçamento, tentativa de pagamento, webhooks, estorno e outbox | PostgreSQL Billing |
| Operações | Reserva/liberação/consumo de estoque e execução, outbox por agregado | PostgreSQL Operações para estoque; DynamoDB para execução, com `TransactWriteItems` para item de execução + outbox |
| Mercado Pago | Processamento externo do pagamento/estorno | Fonte externa; Billing persiste IDs e estados reconciliados, nunca credenciais/dados completos de cartão |

### Envelope de comandos e eventos

Todos os comandos/eventos carregam, no mínimo:

| Campo | Uso |
|---|---|
| `messageId` | UUID único para deduplicação da mensagem |
| `messageType` | Nome de contrato versionado, por exemplo `PaymentApproved.v1` |
| `occurredAtUtc` | Instante UTC de criação/publicação do fato |
| `producer` | Serviço que publicou o contrato |
| `correlationId` | Identificador estável da Saga iniciada pela OS |
| `causationId` | ID do comando/evento que causou a mensagem atual |
| `osId`, `filialId` | Referências necessárias ao fluxo de negócio |
| `idempotencyKey` | Chave estável por efeito/comando; evita duplicar pagamento/reserva/execução |
| `schemaVersion` | Versão explícita do payload |

Payloads contêm referências e dados mínimos necessários. Não incluem segredo, access token, CPF completo, PAN/CVV ou cópia de estado que tenha outro serviço como fonte da verdade. Mudanças incompatíveis criam nova versão; consumidores ignoram/colocam em quarentena tipos/versões desconhecidos e emitem alerta, sem descartar silenciosamente.

### Catálogo inicial de contratos

| Contrato v1 | Produtor | Consumidor(es) | Efeito/resultado |
|---|---|---|---|
| `QuoteRequested.v1` | OS | Billing | Solicita orçamento para `osId`, versão de solicitação e referências dos itens |
| `QuoteReady.v1` | Billing | OS | Publica `quoteId`, versão, valor, moeda e validade para apresentação |
| `QuoteRejected.v1` | Billing | OS | Rejeita solicitação inválida/sem itens suficientes; não cria pagamento |
| `QuoteApprovalRequested.v1` | OS (após ação do cliente) | Billing | Solicita aprovação/rejeição de uma versão exata do orçamento |
| `QuoteApproved.v1` / `QuoteDeclined.v1` | Billing | OS | Registra a decisão sob ownership de Billing |
| `InventoryReservationRequested.v1` | OS | Operações | Reserva itens por OS/filial com chave idempotente |
| `InventoryReserved.v1` / `InventoryReservationRejected.v1` | Operações | OS | Confirma reserva ou informa indisponibilidade; rejeição não altera saldo parcialmente |
| `PaymentCreationRequested.v1` | OS | Billing | Solicita pagamento para orçamento/versão/valor/moeda aprovados |
| `PaymentPending.v1` | Billing | OS | Informa tentativa e referência de pagamento; não significa aprovação |
| `PaymentApproved.v1` / `PaymentDeclined.v1` | Billing | OS | Resultado reconciliado com o Mercado Pago |
| `PaymentOutcomeUnknown.v1` | Billing | OS | Resultado ainda incerto por timeout/erro técnico; exige reconciliação, não é recusa |
| `ExecutionStartRequested.v1` | OS | Operações | Inicia execução para reserva já existente e pagamento aprovado |
| `ExecutionStarted.v1` / `ExecutionStartRejected.v1` | Operações | OS | Confirma início ou rejeita antes de executar trabalho |
| `ExecutionCompleted.v1` / `ExecutionFailed.v1` | Operações | OS | Informa resultado e referências de itens efetivamente consumidos |
| `InventoryReleaseRequested.v1` / `InventoryReleased.v1` | OS | Operações | Libera quantidades não consumidas; idempotente |
| `PaymentRefundRequested.v1` | OS | Billing | Solicita compensação financeira após falha posterior ao pagamento aprovado |
| `PaymentRefunded.v1` / `PaymentRefundPending.v1` / `PaymentRefundFailed.v1` | Billing | OS | Resultado confirmado, pendente ou falha de estorno |

`PaymentApproved` só é publicado após verificação da assinatura/notificação e reconciliação da tentativa no Mercado Pago. Receipt/ack HTTP do webhook indica apenas que Billing recebeu a notificação, não que a cobrança foi liquidada.

### Estados persistidos da Saga

`RequestingQuote` → `AwaitingApproval` → `ReservingInventory` → `CreatingPayment` → `AwaitingPayment` → `ReconcilingPayment` (quando resultado técnico incerto) → `StartingExecution` → `InExecution` → `Completed`.

Terminais alternativos: `Cancelled`, `Compensating`, `CompensationPending`, `Compensated` e `ManualActionRequired`. Cada transição grava versão/estado e outbox na mesma transação PostgreSQL OS. Cada serviço persiste localmente o próprio efeito antes de emitir seu evento de resultado.

### Timeouts e retries iniciais

| Operação | Prazo/política inicial | Ao esgotar ou exceder o prazo |
|---|---|---|
| Aprovação do orçamento | 24 horas desde a publicação; valor configurável | Billing expira a versão e publica `QuoteExpired`; OS encerra sem reserva ou cobrança |
| Reserva de estoque | Resposta em até 30 segundos; lease inicial de 30 minutos antes da criação do pagamento | Enquanto houver tentativa `PaymentPending` ou `PaymentOutcomeUnknown`, OS renova o lease idempotentemente durante a janela de reconciliação. Não liberar automaticamente enquanto o resultado do pagamento for incerto; no limite configurado, pausar em `ManualActionRequired` e manter a reserva até reconciliação/ação auditada |
| Criação/consulta do pagamento | Janela inicial de 30 minutos, alinhada à expiração configurada no produto do Mercado Pago e validada no CARD-39b | Nunca interpretar timeout como recusa; reconciliar consulta/callback pelo mesmo `idempotencyKey`; manter reserva até confirmação de estado terminal ou reconciliação manual |
| Comando/evento transitório | Até 3 tentativas com backoff inicial de 1 s, 5 s e 10 s, depois política de DLQ/manual replay | Alertar, manter estado não terminal e permitir replay idempotente |
| Estorno | Estado `RefundPending` persiste até confirmação final; retry/reconciliação com o mesmo identificador do provedor | Não declarar Saga compensada; após limite operacional, `ManualActionRequired` e alerta até conciliação |

Os valores configuráveis são defaults do MVP. CARD-39b deve confirmar quais expirações e chaves idempotentes o produto/API do Mercado Pago selecionado suporta; não se deve ajustar o estado de negócio para fingir que um timeout externo é resultado terminal.

### Compensações e efeitos não reversíveis

| Falha após etapa | Compensação/ação | Condição para encerrar |
|---|---|---|
| Orçamento rejeitado/expirado antes da reserva | Cancelar Saga; nenhum pagamento/reserva existe | OS registra `Cancelled` |
| Estoque insuficiente antes do pagamento | Recusar início; Billing não cria pagamento | OS registra `Cancelled`/`Rejected` após resultado de Operações |
| Pagamento definitivamente recusado/cancelado | Comandar liberação da reserva | Só encerra após `InventoryReleased`; se a liberação não confirmar, `CompensationPending` |
| Resultado de pagamento desconhecido | Consultar/reconciliar Mercado Pago, renovar a reserva enquanto a tentativa estiver pendente; não cobrar novamente sem idempotency key | `AwaitingPayment`/`ReconcilingPayment` até resultado confirmado; depois aplicar aprovado ou recusado. Se exceder a janela, manter reserva e exigir reconciliação manual, sem liberar estoque por timeout |
| Pagamento aprovado, mas execução não iniciou/trabalho não começou | Solicitar estorno e liberação da reserva como passos compensatórios separados | Só `Compensated` após confirmação de estorno e de liberação; se um lado falhar, `ManualActionRequired` |
| Execução falhou após consumo físico parcial | Operações registra a quantidade realmente consumida e libera somente a parte não utilizada; Billing solicita estorno conforme política financeira aprovada | Saga não inventa reversão de trabalho/peça já consumidos; estados financeiros e operacionais ficam explícitos e falha não resolvida exige intervenção manual |
| Mensagem duplicada/restart de serviço | Deduplicar por `messageId`/`idempotencyKey`, retomar do estado persistido e reemitir outbox pendente | Reprocessamento não repete efeitos confirmados |

Compensação é uma nova ação de negócio, não rollback ACID distribuído. Se um efeito externo ou trabalho físico não puder ser desfeito, o serviço registra o estado real e aciona recuperação manual auditável; não declara sucesso/compensação sem confirmação.

### Outbox, inbox e idempotência

- OS salva transição da ordem/estado da Saga e evento na mesma outbox PostgreSQL.
- Billing salva alteração de orçamento/pagamento e outbox no mesmo PostgreSQL.
- Operações salva estoque e sua outbox na mesma transação PostgreSQL; para execução DynamoDB grava o agregado e item de outbox em `TransactWriteItems`.
- Dispatchers podem publicar mais de uma vez; consumers persistem inbox/deduplication key no store dono antes de confirmar a mensagem.
- Callback do Mercado Pago deduplica por ID de notificação/recurso e associação com tentativa interna. Callback atrasado não regride estado terminal.
- DLQ contém referência/correlation ID e motivo, com reprocessamento auditado; payload sensível é minimizado.

---

## Alternativas consideradas

### Coreografia somente por eventos

**Não recomendada para o primeiro desenho, mas permanece válida:** três serviços e pagamentos externos têm ordem, timeout e compensações relevantes. Sem coordenador explícito, progressão e recuperação podem ficar mais difíceis de explicar e demonstrar. A coreografia reduz a dependência de um coordenador central, mas distribui decisões e recuperação pelos participantes; comparar com os critérios do CARD-36 antes de confirmar.

### Orchestrator como quarto microsserviço

**Não recomendada:** duplicaria estado/infra/repositório. OS já é dono do ciclo da ordem e poderia hospedar a coordenação em seu próprio banco, sem alterar o mínimo de três serviços.

---

## Consequências

### Benefícios esperados se a recomendação for confirmada

- Existe um lugar explícito para inspecionar estado, timeout e decisão da Saga.
- Serviços preservam banco e ownership próprios; OS coordena por contratos, não por acesso cruzado.
- O fluxo reserva estoque antes de cobrar e torna visíveis resultados financeiros incertos.
- O desenho inclui compensações verificáveis, outbox e recuperação após restart.

### Riscos a considerar na avaliação

| Risco | Mitigação |
|---|---|
| OS concentra coordenação e pode acumular lógica | Implementar Saga como módulo interno isolado, com máquina de estados/testes próprios; não mover regras de Billing/Operações para OS. |
| API do Mercado Pago tem estados/limites específicos | Confirmar no CARD-39b o produto selecionado, assinatura de webhook, idempotência, cancelamento/estorno e expiração suportados. |
| Execução pode produzir efeitos físicos irreversíveis | Registrar consumo real e distinguir compensação técnica de recuperação manual/decisão financeira. |
| Retry e DLQ podem deixar Saga parada | Métricas/alertas no CARD-42, DLQ com replay auditado e cenário de falha automatizado no CARD-40. |

---

## Referências

- [Enunciado Tech Challenge — Fase 4](../cards/06-fase4-microsservicos-saga/README.md)
- [ADR-014 — Limites dos microsserviços](ADR-014-limites-microsservicos-fase4.md)
- [ADR-015 — Ownership e infraestrutura](ADR-015-ownership-e-infraestrutura-fase4.md)
- [ADR-016 — Persistência SQL e NoSQL](ADR-016-bancos-sql-nosql-fase4.md)
- [CARD-36 — Contratos e desenho da Saga](../cards/06-fase4-microsservicos-saga/CARD-36-contratos-e-desenho-saga.md)
- [Microservices.io — Saga pattern](https://microservices.io/patterns/data/saga.html)
- [AWS Prescriptive Guidance — Saga orchestration](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-orchestration.html)
- [AWS Prescriptive Guidance — Saga choreography](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-choreography.html)