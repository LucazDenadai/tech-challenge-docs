# ADR-017 — Saga orquestrada pelo serviço OS

**Status:** Aceito — estratégia confirmada pelo time em 2026-10-05; implementação pendente nos CARD-37 a CARD-40
**Data:** 2026-10-05
**Autores:** Time Tech Challenge — Fase 4
**Relaciona-se a:** [ADR-014](ADR-014-limites-microsservicos-fase4.md), [ADR-015](ADR-015-ownership-e-infraestrutura-fase4.md), [ADR-016](ADR-016-bancos-sql-nosql-fase4.md), [CARD-36](../cards/06-fase4-microsservicos-saga/CARD-36-contratos-e-desenho-saga.md)

---

## Contexto

O fluxo da Fase 4 atravessa os serviços OS, Billing e Operações. Cada serviço é dono do próprio banco e não pode consultar tabelas de outro serviço. A Saga precisa acompanhar o progresso, tolerar duplicidade/restart, compensar efeitos já confirmados e demonstrar o tratamento de falha.

O estado geral da OS pertence a OS, tornando-o o local mais direto para hospedar a coordenação sem criar um quarto microsserviço. Billing continua dono do orçamento/pagamento e Operações do estoque/execução. O orchestrator nunca grava diretamente nas bases desses participantes.

---

## Decisão

Adota-se **Saga orquestrada pelo serviço OS**. A coordenação é um módulo interno de OS, persistido na instância PostgreSQL exclusiva de OS junto a uma outbox local. Não cria um quarto serviço e não muda os limites do ADR-014.

### Ordem do fluxo normal

1. OS cria a ordem e a Saga no mesmo commit local, inicialmente `Diagnosing`, e publica `DiagnosisRequested` via outbox. A ordem não traz itens na abertura; eles resultam do diagnóstico, como no fluxo da Fase 3 (`Recebida` → `EmDiagnostico` → `AguardandoAprovacao`).
2. Operações registra o diagnóstico no agregado de execução (DynamoDB) e publica `DiagnosisCompleted` com as peças/serviços necessários e o preço vigente de cada item no seu catálogo (snapshot, conforme emenda do ADR-015), ou `DiagnosisRejected`. O registro diagnosticado não entra na fila de execução até receber `ExecutionStartRequested`.
3. OS publica `QuoteRequested` com os itens e preços do diagnóstico. Billing valida o pedido, calcula o total a partir do snapshot recebido (sem consultar Operações), cria uma versão de orçamento sob seu ownership e publica `QuoteReady` ou `QuoteRejected`.
4. Cliente autorizado aprova ou rejeita uma versão específica do orçamento chamando o Billing via REST (API Gateway, mesmo JWT). Billing é autoridade da aprovação e responde de forma síncrona: sucesso, versão desatualizada (`409`) ou orçamento expirado (`410`). Registrada a decisão, Billing publica `QuoteApproved` ou `QuoteDeclined` via outbox, e OS segue a Saga a partir desse evento.
5. Após aprovação, OS pede a Operações uma reserva de peças por `(filialId, pecaId)`. Se não houver saldo na filial, a Saga é cancelada sem criar pagamento.
6. Após `InventoryReserved`, OS pede a Billing a criação idempotente de uma tentativa de pagamento para a versão/valor/moeda aprovados. Billing cria a tentativa no Mercado Pago e publica `PaymentPending` com a referência do provedor.
7. O webhook é recebido e validado por Billing. Só depois de reconciliar com o provedor Billing publica `PaymentApproved`, `PaymentDeclined` ou o estado técnico de resultado desconhecido.
8. Após `PaymentApproved`, OS comanda Operações a iniciar a execução reservada. A execução diagnosticada entra na fila e registra o progresso do reparo no DynamoDB; o estoque permanece no PostgreSQL de Operações.
9. Operações conclui o trabalho, consome da reserva a quantidade efetivamente utilizada e publica `ExecutionCompleted`. OS registra o status/histórico final da ordem e encerra a Saga como `Completed`.

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
| `DiagnosisRequested.v1` | OS | Operações | Solicita diagnóstico do veículo da OS na filial |
| `DiagnosisCompleted.v1` / `DiagnosisRejected.v1` | Operações | OS | Informa peças/serviços necessários com `itemId`, tipo, quantidade, preço unitário, moeda e `priceSnapshotAtUtc`, ou rejeita o pedido (por exemplo, filial inválida); não reserva estoque |
| `QuoteRequested.v1` | OS | Billing | Solicita orçamento para `osId`, versão de solicitação e itens do diagnóstico |
| `QuoteReady.v1` | Billing | OS | Publica `quoteId`, versão, valor, moeda e validade para apresentação |
| `QuoteRejected.v1` | Billing | OS | Rejeita solicitação inválida/sem itens suficientes; não cria pagamento |
| `QuoteApproved.v1` / `QuoteDeclined.v1` | Billing | OS | Registra a decisão sob ownership de Billing |
| `QuoteExpired.v1` | Billing | OS | Versão do orçamento expirou sem decisão no prazo; OS encerra sem reserva ou cobrança |
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

O preço do orçamento é o snapshot do `DiagnosisCompleted`: mudanças posteriores no catálogo de Operações não alteram orçamentos já emitidos, e o replay da mesma mensagem gera o mesmo valor. Billing rejeita com `QuoteRejected` itens sem preço, com preço não positivo ou sem moeda. Revisão do diagnóstico depois do orçamento (por exemplo, peça adicional descoberta durante o reparo) fica fora do MVP; quando entrar, gera novo diagnóstico, nova versão de orçamento e nova aprovação.

### Estados persistidos da Saga

`Diagnosing` → `RequestingQuote` → `AwaitingApproval` → `ReservingInventory` → `CreatingPayment` → `AwaitingPayment` → `ReconcilingPayment` (quando resultado técnico incerto) → `StartingExecution` → `InExecution` → `Completed`.

Estados de desvio não terminais: `ReleasingReservation`, `Compensating`, `CompensationPending` e `ManualActionRequired`. Estados terminais: `Completed`, `Cancelled` e `Compensated`. Cada transição grava versão/estado e outbox na mesma transação PostgreSQL OS. Cada serviço persiste localmente o próprio efeito antes de emitir seu evento de resultado.
### Status da OS visível ao clienteO OS mantém um status de negócio derivado do estado da Saga. Ele é gravado na mesma transação da transição da Saga; não há escrita independente de status.| Estados da Saga | Status da OS ||---|---|| `Diagnosing` | `EmDiagnostico` || `RequestingQuote`, `AwaitingApproval` | `AguardandoAprovacao` || `ReservingInventory`, `CreatingPayment`, `AwaitingPayment`, `ReconcilingPayment` | `AguardandoPagamento` || `StartingExecution`, `InExecution` | `EmExecucao` || `Completed` | `Finalizada` || `Cancelled`, `Compensated` | `Cancelada` || `ReleasingReservation`, `Compensating`, `CompensationPending`, `ManualActionRequired` | Mantém o status anterior; o detalhe fica no histórico |`Entregue` é uma transição manual do Atendente a partir de `Finalizada`, fora da Saga. O status `Recebida` da Fase 3 deixa de existir, porque a OS nasce solicitando diagnóstico.

### Timeouts e retries iniciais

| Operação | Prazo/política inicial | Ao esgotar ou exceder o prazo |
|---|---|---|
| Aprovação do orçamento | 24 horas desde a publicação; valor configurável | Billing expira a versão e publica `QuoteExpired`; OS encerra sem reserva ou cobrança |
| Reserva de estoque | Resposta em até 30 segundos; no MVP a reserva não expira e permanece até a Saga liberá-la ou consumi-la | Não liberar automaticamente enquanto o resultado do pagamento for incerto. Lease com renovação fica como evolução (ver [Escopo do MVP](#escopo-do-mvp)) |
| Criação/consulta do pagamento | Janela inicial de 30 minutos, alinhada à expiração configurada no produto do Mercado Pago e validada no CARD-39b | Nunca interpretar timeout como recusa; reconciliar consulta/callback pelo mesmo `idempotencyKey`; manter reserva até confirmação de estado terminal ou reconciliação manual |
| Comando/evento transitório | Até 3 tentativas com backoff inicial de 1 s, 5 s e 10 s, depois política de DLQ/manual replay | Alertar, manter estado não terminal e permitir replay idempotente |
| Estorno | Estado `RefundPending` persiste até confirmação final; retry/reconciliação com o mesmo identificador do provedor | Não declarar Saga compensada; após limite operacional, `ManualActionRequired` e alerta até conciliação |

Os valores configuráveis são defaults do MVP. CARD-39b deve confirmar quais expirações e chaves idempotentes o produto/API do Mercado Pago selecionado suporta; não se deve ajustar o estado de negócio para fingir que um timeout externo é resultado terminal.

### Compensações e efeitos não reversíveis

| Falha após etapa | Compensação/ação | Condição para encerrar |
|---|---|---|
| Diagnóstico rejeitado | Cancelar Saga; nenhum orçamento, reserva ou pagamento existe | OS registra `Cancelled` |
| Orçamento rejeitado/expirado antes da reserva | Cancelar Saga; nenhum pagamento/reserva existe. O diagnóstico registrado em Operações permanece como histórico e não entra na fila de execução | OS registra `Cancelled` |
| Estoque insuficiente antes do pagamento | Recusar início; Billing não cria pagamento | OS registra `Cancelled` após resultado de Operações |
| Pagamento definitivamente recusado/cancelado | Comandar liberação da reserva | Só encerra após `InventoryReleased`; se a liberação não confirmar, `CompensationPending` |
| Resultado de pagamento desconhecido | Job de Billing consulta/reconcilia o Mercado Pago pela mesma tentativa, mantendo a reserva; não cobrar novamente sem idempotency key | `AwaitingPayment`/`ReconcilingPayment` até resultado confirmado; depois aplicar aprovado ou recusado. Se exceder a janela, manter reserva e exigir reconciliação manual, sem liberar estoque por timeout |
| Pagamento aprovado, mas execução não iniciou/trabalho não começou | Solicitar estorno e liberação da reserva como passos compensatórios separados | Só `Compensated` após confirmação de estorno e de liberação; se um lado falhar, `ManualActionRequired` |
| Execução falhou após consumo físico parcial | Operações registra a quantidade realmente consumida como baixa e libera somente a parte não utilizada; OS solicita estorno **total** ao Billing (a oficina absorve o custo das peças consumidas) | Só `Compensated` após confirmação de estorno e de liberação; se um lado falhar, `ManualActionRequired`. A Saga não afirma que o trabalho/peça consumidos foram desfeitos |
| Mensagem duplicada/restart de serviço | Deduplicar por `messageId`/`idempotencyKey`, retomar do estado persistido e reemitir outbox pendente | Reprocessamento não repete efeitos confirmados |

Compensação é uma nova ação de negócio, não rollback ACID distribuído. Se um efeito externo ou trabalho físico não puder ser desfeito, o serviço registra o estado real e aciona recuperação manual auditável; não declara sucesso/compensação sem confirmação.

### Escopo do MVP

O PDF exige "rollback e compensação no caso de falha em qualquer etapa". O MVP implementa um caminho de falha para **cada** etapa da tabela acima, sempre com a compensação mais simples: cancelamento quando nada foi efetivado, liberação de reserva, estorno total. Ficam documentados como evolução, sem implementação no MVP:

- Lease da reserva de estoque com expiração e renovação durante a reconciliação do pagamento.
- Replay auditado a partir de `ManualActionRequired` via endpoint administrativo; no MVP o estado é persistido e alertado, e a saída é manual.
- Estorno parcial proporcional ao trabalho/peças já consumidos quando a execução falha.
- Reconciliação dedicada de estorno pendente; no MVP vale a política genérica de retry/DLQ.
- Revisão de diagnóstico/orçamento após aprovação.

Testes: o fluxo completo (caminho feliz) é o cenário BDD obrigatório do PDF; cada linha da tabela de compensações tem teste de integração no CARD-40. O vídeo demonstra o caminho feliz e ao menos as falhas "pagamento recusado" e "execução falhou".

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

**Não escolhida, mas permanece válida:** três serviços e pagamentos externos têm ordem, timeout e compensações relevantes. Sem coordenador explícito, progressão e recuperação ficam mais difíceis de explicar e demonstrar. A coreografia reduz a dependência de um coordenador central, mas distribui decisões e recuperação pelos participantes.

### Orchestrator como quarto microsserviço

**Rejeitada:** duplicaria estado/infra/repositório. OS já é dono do ciclo da ordem e poderia hospedar a coordenação em seu próprio banco, sem alterar o mínimo de três serviços.

---

## Consequências

### Positivas

- Existe um lugar explícito para inspecionar estado, timeout e decisão da Saga.
- Serviços preservam banco e ownership próprios; OS coordena por contratos, não por acesso cruzado.
- O fluxo reserva estoque antes de cobrar e torna visíveis resultados financeiros incertos.
- O desenho inclui compensações verificáveis, outbox e recuperação após restart.

### Negativas e riscos

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