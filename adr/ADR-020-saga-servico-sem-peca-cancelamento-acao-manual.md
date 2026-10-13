# ADR-020 — Saga: serviço sem peça, cancelamento antes do reparo e resolução manual

**Status:** Aceito
**Data:** 2026-10-13
**Autores:** Time Tech Challenge — Fase 4
**Relaciona-se a:** [ADR-015](ADR-015-ownership-e-infraestrutura-fase4.md), [ADR-017](ADR-017-saga-orquestrada-os-fase4.md), [ADR-018](ADR-018-contratos-assincronos-asyncapi-fase4.md), [ADR-019](ADR-019-integracao-mercado-pago-fase4.md), [CARD-40a](../cards/06-fase4-microsservicos-saga/CARD-40a-orquestrador-saga-os.md), [CARD-40b](../cards/06-fase4-microsservicos-saga/CARD-40b-compensacoes-e-falhas.md)

---

## Contexto

O CARD-40a implementou o fluxo normal da Saga e deixou três casos sem caminho:

1. **Serviço sem peça.** Uma OS pode ter só serviços: troca de óleo com o óleo do cliente, balanceamento, alinhamento. O contrato exige reserva com ao menos uma peça (`InventoryReservationRequested.items`, `minItems: 1`) e o início da execução exige uma reserva (`ExecutionStartRequested.reservationId`). Hoje a Saga para em `ManualActionRequired`.
2. **Cancelamento durante a execução.** O cancelamento manual da OS (CARD-37b) não compensa nada, e o AsyncAPI não tem comando para cancelar uma execução em Operações ([CARD-38b](../cards/06-fase4-microsservicos-saga/CARD-38b-fila-execucao.md)). O cliente tem direito de desistir, mas parte do trabalho pode já ter sido feita.
3. **Saída de `ManualActionRequired`.** O ADR-017 exige recuperação manual auditável, mas não define quem resolve nem como.

Restrições:

- O OS não sabe se o reparo começou. A passagem de `NaFila` para `EmReparo` existe só em Operações (emenda "Progresso da execução" do ADR-015); para o OS, as duas são `EmExecucao`.
- O Billing só faz estorno total ([CARD-39b](../cards/06-fase4-microsservicos-saga/CARD-39b-mercado-pago-webhook.md)).
- Tornar opcional um campo obrigatório é mudança incompatível pelo ADR-018, que pede um canal `.v2`. Nenhum serviço da Fase 4 teve deploy ainda (CARD-41).

---

## Decisão

### Mudança incompatível na v1 antes do primeiro deploy

Até o primeiro deploy de um serviço da Fase 4, um canal `.v1` pode receber mudança incompatível, se as três condições valerem:

- A mudança está registrada em ADR.
- A spec em `tech-challenge-docs` e as cópias nos repositórios mudam no mesmo ciclo.
- Produtor e todos os consumidores do canal são atualizados antes do próximo deploy.

Depois do primeiro deploy, vale a regra do ADR-018 sem exceção. A exceção existe porque não há consumidor em produção para quebrar, e um `.v2` agora só criaria dois canais para o mesmo uso.

### Serviço sem peça

- `ExecutionStartRequested.reservationId` passa a ser opcional (mudança incompatível pela exceção acima). Ausente significa orçamento sem peças.
- Na Saga, `QuoteApproved` de orçamento sem peças vai de `AwaitingApproval` direto para `CreatingPayment`, sem pedir reserva.
- Operações aceita início sem `reservationId` só quando o diagnóstico não tem peças. Diagnóstico com peças e sem reserva recebe `ExecutionStartRejected`. A conclusão sem reserva não consome estoque.
- Nas compensações, sem reserva não há liberação: só o estorno.

### Cancelamento: permitido antes do reparo, recusado em reparo

- Novos canais (mudança compatível): `ExecutionCancelRequested.v1` (OS → Operações) e `ExecutionCancelled.v1` / `ExecutionCancelRejected.v1` (Operações → OS).
- **Operações decide**, porque só ela sabe se o reparo começou. Cancela execução `Diagnosticada` ou `NaFila`; recusa `EmReparo` ou já encerrada. O cancelamento é idempotente pela `idempotencyKey`.
- Cancelada a execução, a Saga compensa: estorno total e liberação da reserva. Não há multa, porque nenhum trabalho foi feito.
- Recusado (reparo já começou), a Saga volta a `InExecution` e o motivo fica no histórico.
- O cancelamento manual pela API vira um pedido. A rota responde `204` quando nada precisa ser desfeito e `202` quando a Saga precisa compensar ou esperar a resposta de um participante. O resultado aparece no status e no histórico.
- Novo estado da Saga, de desvio: `CancellingExecution`. O status da OS continua `EmExecucao` até a resposta.

### Resolução de `ManualActionRequired` pelo Admin

- Rota `POST /os/ordens-servico/{id}/saga/resolucao`, só `Admin`, com `acao` e `motivo` obrigatórios. A resolução fica no histórico da Saga com o motivo e o usuário.
- **`Cancelar`:** compensa conforme os efeitos já confirmados (estorno do pagamento aprovado, liberação da reserva existente) e termina em `Compensated` ou `Cancelled`.
- **`Retomar`:** volta ao estado em que a Saga parou e reenvia o comando dessa etapa com a mesma `idempotencyKey`. Vale quando a causa foi corrigida fora do sistema, como um participante que ficou fora do ar além do prazo.
- Pagamento aprovado com valor ou moeda divergentes só aceita `Cancelar`: retomar liberaria a execução sobre um valor que não foi aprovado.

---

## Alternativas consideradas

### Canal `.v2` para o início sem reserva

**Não escolhida:** segue o ADR-018 à risca, mas cria dois canais para o mesmo comando sem nenhum consumidor antigo em produção. A exceção vale só até o primeiro deploy.

### Reserva vazia

**Não escolhida:** Operações aceitaria reserva sem itens e o contrato de início não mudaria. Criaria registros de estoque sem significado e esconderia a regra "não há peça" atrás de uma reserva fictícia.

### Cancelamento em qualquer etapa da execução, com estorno total

**Não escolhida:** a oficina absorveria trabalho e peças de um reparo em andamento por decisão do cliente, não por falha dela. O ADR-017 aceita esse custo só quando a execução falha.

### Multa ou estorno parcial

**Evolução:** exige uma política escrita (valor, estados, peças já usadas) e estorno parcial no Billing, que hoje só faz estorno total. Fica fora do MVP.

### OS decidir o cancelamento sozinho

**Não escolhida:** o OS não sabe se o reparo começou. Responder `422` pelo status `EmExecucao` recusaria cancelamentos válidos de execuções ainda na fila.

---

## Consequências

### Positivas

- OS só de serviço passa a ter caminho completo.
- O cliente pode desistir antes do reparo sem custo, e a decisão fica com quem tem a informação (Operações).
- `ManualActionRequired` tem dono e rota, com registro auditável.

### Negativas e riscos

| Risco | Mitigação |
|---|---|
| Exceção de versionamento usada depois do deploy | A exceção cita o primeiro deploy (CARD-41) como fim; depois dele, só `.v2`. |
| Cancelamento passa a ser assíncrono | `202` e status/histórico mostram o andamento; o atendente não recebe falso sucesso. |
| Corrida: o reparo começa enquanto o pedido de cancelamento viaja | Operações é a autoridade e recusa; a Saga volta a `InExecution`. |
| Retomada mal usada | Só `Admin`, motivo obrigatório, e proibida para pagamento divergente. |
