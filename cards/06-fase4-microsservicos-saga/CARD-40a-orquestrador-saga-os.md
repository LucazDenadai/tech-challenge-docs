# CARD-40a — Orquestrador da Saga no OS: fluxo normal

**Tipo:** Implementação / Integração distribuída
**Status:** To Do — escopo e decisões definidos (2026-10-13)
**Depende de:** CARD-36, CARD-37b, CARD-38b, CARD-39b
**Bloqueia:** CARD-40b, CARD-40c
**Repositório alvo:** `tech-challenge-os` ([ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md))
**Decisão arquitetural:** [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md), [ADR-018](../../adr/ADR-018-contratos-assincronos-asyncapi-fase4.md) e [ADR-019](../../adr/ADR-019-integracao-mercado-pago-fase4.md); contratos do CARD-36

---

## Contexto

Os três serviços já existem, mas não conversam. Operações e Billing processam comandos e publicam eventos; o OS só grava os eventos recebidos na inbox. Falta o orquestrador do ADR-017: a Saga persistida no OS, a outbox e as transições que levam a OS da abertura até a conclusão. Este card entrega o caminho normal e as rejeições que encerram a Saga sem compensação. Compensações e falhas ficam no [CARD-40b](CARD-40b-compensacoes-e-falhas.md); o ambiente integrado e o BDD, no [CARD-40c](CARD-40c-ambiente-integrado-e-bdd.md).

## Ponto de partida (2026-10-13)

- **OS ([CARD-37b](CARD-37b-api-e-contratos-os.md)):** consumidor dos 21 canais que o OS recebe, com validação do envelope, deduplicação na tabela `InboxMensagens` (payload em `jsonb`) e DLQ. O consumidor só registra; não há efeito de negócio, outbox nem publicador. `AbrirOrdemServicoUseCase` cria a OS em `EmDiagnostico` sem publicar nada.
- **Status da OS:** `StatusOrdemServico` já segue o mapeamento do ADR-017. A API só tem as transições manuais `entrega` e `cancelamento`.
- **Operações ([CARD-38b](CARD-38b-fila-execucao.md)):** consome `DiagnosisRequested`, `InventoryReservationRequested`, `InventoryReleaseRequested` e `ExecutionStartRequested`, com efeito local e resultado pela outbox.
- **Billing ([CARD-39a](CARD-39a-orcamento-aprovacao.md), [CARD-39b](CARD-39b-mercado-pago-webhook.md)):** consome `QuoteRequested`, `PaymentCreationRequested` e `PaymentRefundRequested`. `PaymentCreationRequested` exige `payerEmail`.
- **Cliente:** `Cliente.Email` existe, mas o cadastro não valida o campo.

## Decisões de implementação (2026-10-13)

- **Agregado `SagaOrdemServico`:** um por OS, na base PostgreSQL do OS. `Id` = `correlationId` da Saga. Campos: `OsId` (único), `FilialId`, `ClienteId`, `Estado`, referências recebidas dos participantes (`QuoteId`, `QuoteVersion`, valor e moeda aprovados, `PaymentId`, itens do diagnóstico), `UltimaMensagemId` e datas. Concorrência otimista pelo `xmin`, como em Billing.
- **Estados:** os do ADR-017. Este card usa `Diagnosing`, `RequestingQuote`, `AwaitingApproval`, `ReservingInventory`, `CreatingPayment`, `AwaitingPayment`, `StartingExecution`, `InExecution`, `Completed` e `Cancelled`. Os estados de desvio (`ReconcilingPayment`, `ReleasingReservation`, `Compensating`, `CompensationPending`, `ManualActionRequired`, `Compensated`) entram no enum agora, mas as transições ficam no CARD-40b.
- **Status da OS derivado:** cada transição da Saga atualiza o status da OS e grava o histórico na mesma transação, pela tabela do ADR-017. Não há escrita de status fora da Saga, exceto as manuais `entrega` e `cancelamento` do CARD-37b.
- **Abertura:** `AbrirOrdemServicoUseCase` cria a OS, a Saga em `Diagnosing` e `DiagnosisRequested` na outbox em um só commit.
- **Processamento das mensagens:** o consumidor passa a aplicar o efeito na mesma transação da inbox, como em Billing e Operações: inbox, transição da Saga, status da OS e comando seguinte na outbox. A deduplicação por `messageId` não muda.
- **Transições deste card:**

  | Estado atual | Mensagem | Novo estado | Comando publicado |
  |---|---|---|---|
  | `Diagnosing` | `DiagnosisCompleted` | `RequestingQuote` | `QuoteRequested` (itens e preços do diagnóstico, `clienteId`) |
  | `Diagnosing` | `DiagnosisRejected` | `Cancelled` | — |
  | `RequestingQuote` | `QuoteReady` | `AwaitingApproval` | — |
  | `RequestingQuote` | `QuoteRejected` | `Cancelled` | — |
  | `AwaitingApproval` | `QuoteApproved` | `ReservingInventory` | `InventoryReservationRequested` |
  | `AwaitingApproval` | `QuoteDeclined` ou `QuoteExpired` | `Cancelled` | — |
  | `ReservingInventory` | `InventoryReserved` | `CreatingPayment` | `PaymentCreationRequested` (valor, moeda e versão aprovados; `payerEmail`) |
  | `ReservingInventory` | `InventoryReservationRejected` | `Cancelled` | — |
  | `CreatingPayment` | `PaymentPending` | `AwaitingPayment` | — |
  | `CreatingPayment` ou `AwaitingPayment` | `PaymentApproved` | `StartingExecution` | `ExecutionStartRequested` |
  | `StartingExecution` | `ExecutionStarted` | `InExecution` | — |
  | `InExecution` | `ExecutionCompleted` | `Completed` | — |

  `PaymentApproved` também vale em `CreatingPayment`, porque a ordem de entrega entre canais diferentes não é garantida: o `PaymentPending` pode chegar depois.
- **Mensagem fora do fluxo:** mensagem válida que não se aplica ao estado atual (atrasada, de estado terminal ou de outra etapa) é confirmada sem mudar estado, com log de aviso e métrica `os_saga_mensagens{resultado="ForaDoFluxo"}`. Ela não vai para a DLQ, porque não está malformada. Evento cujo `osId` não tem Saga vai para a DLQ com motivo.
- **Mensagens do CARD-40b:** `PaymentDeclined`, `PaymentOutcomeUnknown`, `ExecutionStartRejected`, `ExecutionFailed`, `InventoryReleased` e os eventos de estorno ficam registrados na inbox e contam como `ForaDoFluxo` até o CARD-40b.
- **E-mail do pagador:** o `payerEmail` vem do `Cliente.Email` no momento do pedido de pagamento. O cadastro e a atualização do cliente passam a exigir e-mail válido, e a abertura da OS recusa (`422`) cliente sem e-mail, para a Saga não parar no meio. O e-mail só trafega no comando (ADR-019).
- **Outbox e publicador:** tabela `OutboxMensagens` e despachante com confirmação do broker, copiados do padrão de Billing (o código é copiado, não compartilhado, como no CARD-39a). Exchange fanout por canal, declarada pelo produtor. O `traceparent` e o `correlationId` seguem no comando.
- **`idempotencyKey` dos comandos:** derivada da Saga e da etapa, por exemplo `os:<osId>:quote:1`, `os:<osId>:reservation`, `os:<osId>:payment:<quoteVersion>` e `os:<osId>:execution`. Um replay do mesmo evento gera o mesmo comando com a mesma chave; o participante deduplica.
- **Consulta:** `GET /os/ordens-servico/{id}/saga` (`Admin`, `Atendente`) devolve estado, referências e histórico de transições, sem dados do cliente.
- **Fora deste card:** compensações, pagamento desconhecido, timeouts, cancelamento manual pela Saga e cancelamento de execução ([CARD-40b](CARD-40b-compensacoes-e-falhas.md)); broker único e BDD ([CARD-40c](CARD-40c-ambiente-integrado-e-bdd.md)).

## Critérios de aceite

- [ ] A abertura da OS cria a Saga e publica `DiagnosisRequested` pela outbox no mesmo commit.
- [ ] Cada transição da tabela grava estado da Saga, status e histórico da OS e o comando seguinte em uma transação.
- [ ] O estado da Saga sobrevive a restart e pode ser consultado pela API do OS, sem acesso aos bancos de Billing e Operações.
- [ ] Mensagem duplicada não gera segunda transição nem segundo comando; replay gera o mesmo `idempotencyKey`.
- [ ] Mensagem válida fora do fluxo não muda estado e fica visível em log e métrica.
- [ ] Rejeição de diagnóstico, orçamento ou reserva encerra a Saga em `Cancelled` sem pedir pagamento.
- [ ] `PaymentCreationRequested` leva o valor, a moeda e a versão aprovados e o `payerEmail` do cliente.
- [ ] Os comandos publicados passam no teste de contrato contra o AsyncAPI.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Orquestrar o fluxo normal da OS

  Cenário: Abrir OS inicia a Saga
    Dado um cliente com e-mail e um veículo na filial
    Quando o atendente abre a OS
    Então a Saga fica em Diagnosing e a OS em EmDiagnostico
    E DiagnosisRequested é publicado uma vez

  Cenário: Avançar do diagnóstico ao pagamento
    Dado uma Saga em Diagnosing
    Quando chegam DiagnosisCompleted, QuoteReady, QuoteApproved e InventoryReserved
    Então a Saga fica em CreatingPayment e a OS em AguardandoPagamento
    E PaymentCreationRequested leva o valor aprovado e o e-mail do cliente

  Cenário: Concluir após pagamento aprovado
    Dado uma Saga em AwaitingPayment
    Quando chegam PaymentApproved, ExecutionStarted e ExecutionCompleted
    Então a Saga termina em Completed e a OS em Finalizada

  Cenário: Cancelar quando o cliente recusa o orçamento
    Dado uma Saga em AwaitingApproval
    Quando chega QuoteDeclined
    Então a Saga termina em Cancelled e a OS em Cancelada
    E nenhum comando de reserva ou pagamento é publicado

  Cenário: Ignorar evento repetido
    Dado que QuoteApproved já foi processado
    Quando a mesma mensagem chega de novo
    Então a Saga não muda
    E InventoryReservationRequested não é publicado de novo
```

## Passos

1. Domínio: agregado `SagaOrdemServico`, estados e transições, com testes unitários da tabela.
2. Persistência: tabelas da Saga e da outbox, migration e repositórios.
3. Abertura: Saga e `DiagnosisRequested` no commit da OS; e-mail obrigatório no cliente.
4. Processamento: caso de uso que aplica a mensagem da inbox à Saga na mesma transação.
5. Publicação: despachante da outbox com confirmação do broker.
6. API de consulta da Saga e métricas.
7. Testes de integração pelo broker (Testcontainers) e README/OpenAPI.

## Evidências

- Testes unitários da máquina de estados e de idempotência.
- Testes de integração com PostgreSQL e RabbitMQ: abertura → conclusão com os participantes simulados por mensagens, duplicata, fora do fluxo e restart.
- Teste de contrato dos comandos publicados contra o AsyncAPI.
