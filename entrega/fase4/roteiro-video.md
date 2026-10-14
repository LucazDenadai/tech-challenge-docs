# Roteiro do vídeo — Tech Challenge Fase 4

Vídeo de até 15 minutos ([CARD-43b](../../cards/06-fase4-microsservicos-saga/CARD-43b-roteiro-video.md)). Meta: **13 min**, com 2 min de folga.

O PDF da Fase 4 pede quatro demonstrações. Cada uma tem um bloco próprio:

| # | Pedido do enunciado | Bloco | Evidência na tela |
|---|---|---|---|
| 1 | Fluxo completo de uma OS passando pelos microsserviços | 3 | Pasta 3 do Postman e a Saga em `Completed` com as transições |
| 2 | Execução do Saga Pattern e tratamento de falhas | 4 | Pasta 4: estorno, liberação da reserva e `Compensated` |
| 3 | Deploy automatizado de pelo menos um microsserviço com validação de testes | 5 | PR com `build-e-testes` (gate ≥80%) e `deploy-kind`; merge e imagem `sha-<commit>` |
| 4 | Monitoramento e rastreamento dos fluxos distribuídos | 6 | Dashboard da Saga, trace no Jaeger, logs no Loki e um alerta disparando |

---

## Antes de gravar

### Ambiente (cerca de 15 min antes)

```sh
# tech-challenge-infra-k8s, com os três serviços lado a lado na main
./kind/derrubar.sh && ./kind/subir.sh      # cluster novo, cerca de 10 min
```

1. Preencher no Postman o environment **Tech Challenge — Fase 4 no Kind local** (`senha_demo` e `jwt_key`, ver `postman/README.md`). **Não mostrar esses valores no vídeo.**
2. Rodar a collection inteira uma vez, fora da gravação. Isso aquece os serviços e já põe dados nos dashboards.
3. Deixar abertas, nesta ordem, as abas:
   - diagrama geral (PDF de entrega ou `diagramas/`);
   - Postman com a collection;
   - GitHub: um PR já mergeado de um serviço (sugestão: `tech-challenge-billing` PR #12) e a aba **Actions** do mesmo repositório;
   - Grafana http://localhost:13000, dashboard **Saga da OS**;
   - Jaeger http://localhost:16690;
   - Prometheus http://localhost:19090/alerts;
   - RabbitMQ http://localhost:15680, já logado (usuário e senha em `plataforma-segredos`; não mostrar).
4. Fechar terminais com senhas, notificações e abas pessoais.

### Checklist

- [ ] `kubectl -n oficina-mecanica get pods`: tudo `Running`.
- [ ] Collection verde uma vez.
- [ ] Nenhum token, senha ou segredo visível nas abas.
- [ ] Cronômetro à vista.

---

## Bloco 1 — Abertura (0:00 a 0:40)

**Tela:** capa do PDF ou README do `tech-challenge-docs`.

**Falar:**
> "Sou o Lucaz Denadai, RM37376. Nesta fase, a oficina mecânica virou três microsserviços — OS, Billing e Operações — coordenados por uma Saga orquestrada. Vou mostrar o fluxo completo de uma OS, uma falha com compensação, o deploy automatizado com testes e a observabilidade do fluxo distribuído."

## Bloco 2 — Arquitetura (0:40 a 2:30)

**Tela:** diagrama geral.

**Falar, apontando no diagrama:**
- **OS:** abre a OS, guarda status e histórico e **orquestra a Saga** (ADR-017). Banco PostgreSQL próprio.
- **Billing:** orçamento, aprovação e pagamento Pix pelo Mercado Pago (ADR-019). Banco PostgreSQL próprio.
- **Operações:** estoque em PostgreSQL e execução em **DynamoDB**. É o banco NoSQL exigido (ADR-016).
- **Comunicação:** comandos e eventos pelo RabbitMQ, com contrato AsyncAPI. Cada serviço grava a mensagem na **outbox** na mesma transação do dado, e a **inbox** descarta repetidas. Nenhum serviço lê o banco de outro.
- **Por que orquestração:** um lugar só sabe em que etapa a OS está, quais compensações faltam e quando parar para ação manual.
- **Sem AWS nesta fase (ADR-021):** o Kubernetes é o Kind, no CI e na demonstração. Sem a Lambda de CPF, o token de cliente é gerado no mesmo formato.

## Bloco 3 — Fluxo completo (2:30 a 5:15) · pedido 1

**Tela:** Postman.

1. Rodar as pastas **1. Login** e **2. Preparação** no runner. Comentar: o token do funcionário vem do OS e vale nos três serviços.
2. Rodar a pasta **3. Fluxo completo até Completed**. Enquanto roda, narrar os passos pelos nomes:
   - 3.2: o atendente abre a OS, e a Saga pede o diagnóstico a Operações;
   - 3.4: o mecânico diagnostica 2 litros de óleo e a troca;
   - 3.6 e 3.7: o **cliente** aprova o orçamento no Billing, com o token no formato da Lambda;
   - 3.8 e 3.9: Operações reserva o estoque, o Billing cria o Pix, e o simulador faz o papel do cliente pagando;
   - 3.11 e 3.12: o mecânico inicia e conclui o reparo;
   - 3.13: a Saga termina em `Completed`.
3. Abrir a resposta do **3.20 Saga da OS (Admin)** e rolar as `transicoes`: cada mudança de estado com a mensagem que a causou. Mostrar o **3.15 Histórico** (`AguardandoAprovacao → AguardandoPagamento → EmExecucao → Finalizada`) e o **3.17** (estoque baixou 2).
4. **Copiar o `correlationId`** da resposta do 3.20 para o bloco 6.

**Falar:** "Os requests 'Aguardar' existem porque os passos andam por mensagem: a API responde na hora, e a Saga avança em segundo plano."

## Bloco 4 — Saga e tratamento de falhas (5:15 a 7:45) · pedido 2

**Tela:** Postman.

1. Rodar a pasta **4. Compensação: falha no reparo depois do pagamento**.
2. Quando chegar ao **4.12**, narrar: "o pagamento já foi aprovado e a peça já está reservada. Agora o reparo falha."
3. Mostrar o resultado:
   - **4.13:** a Saga termina em `Compensated`;
   - **4.15:** o pagamento ficou `Estornado` no Billing;
   - **4.16:** a reserva foi liberada e o estoque voltou ao valor inicial.
4. Rodar a pasta **5. Pagamento recusado** e mostrar só o resultado: `Cancelled`, pagamento `Recusado`, reserva liberada.

**Falar sobre o que não aparece na tela:**
- comandos com `idempotencyKey`: repetir um comando não duplica o efeito;
- prazo e reenvio: sem resposta em 5 minutos, a Saga reenvia; depois de 3 reenvios, para em **ação manual**, e o Admin decide Cancelar ou Retomar (ADR-020);
- mensagem fora do contrato vai para a DLQ com o motivo.

## Bloco 5 — Deploy automatizado com testes (7:45 a 10:00) · pedido 3

**Tela:** GitHub.

1. Abrir o PR do serviço escolhido e a aba **Checks**:
   - `build-e-testes`: build, testes unitários e de integração (Testcontainers), **gate de cobertura de linhas e branches ≥80%** e verificação do contrato AsyncAPI;
   - `deploy-kind`: sobe um Kind no runner, implanta **só este serviço**, espera o rollout e o `/ready`.
2. Mostrar no resumo do job o relatório de cobertura (linhas e branches).
3. Mostrar **Settings → Branches**: a `main` exige PR e o check verde, também para administradores.
4. Abrir a execução da `main` depois do merge: o job `imagem` publica `ghcr.io/...:sha-<commit>`, e o `deploy-kind` baixa exatamente essa imagem. Não existe tag `latest`.
5. No OS, mostrar o workflow **BDD ponta a ponta**: o cenário obrigatório em Gherkin, com os três serviços de verdade.

**Falar:** "Cada serviço tem o próprio pipeline e implanta só a si mesmo. Um PR que derruba a cobertura ou quebra o rollout não entra na main."

## Bloco 6 — Monitoramento e rastreamento (10:00 a 12:40) · pedido 4

1. **Grafana, dashboard Saga da OS:** Sagas por estado, concluídas e compensadas na janela, duração, reenvios e DLQ. "O estado do fluxo vem das métricas da Saga, não só de o serviço estar no ar."
2. **Logs da Saga:** colar o `correlationId` do bloco 3 na variável `correlationId` do dashboard. Aparecem os logs dos três serviços. Abrir uma linha e clicar no link **TraceId**.
3. **Jaeger:** Service `os`, Tags `correlation_id=<id>`, Find Traces. Abrir o trace maior: os spans de `os`, `billing` e `operacoes` em sequência, com publicação e consumo das mensagens. "Cada ação humana abre um trace; a tag `correlation_id` junta todos os traces da mesma Saga."
4. **Alerta ao vivo (cerca de 30 s):**
   - no RabbitMQ, Queues → `os.saga-os.quote-ready.v1` → **Publish message**, payload `{"mensagem":"fora do contrato"}`;
   - o OS rejeita a mensagem e a manda para `os.saga-os.quote-ready.v1.dlq`;
   - no Prometheus `/alerts`, `DlqCrescendo` fica **firing**. Abrir o `runbook_url` e mostrar a seção do runbook;
   - opcional: fazer o **Purge** da DLQ e mostrar o alerta sumindo.
5. **Falar:** "É a stack da Fase 3 — Prometheus, Grafana, Loki e Jaeger — no Kind, sem custo (ADR-022). Os serviços exportam por OpenTelemetry. Se a telemetria cair, o fluxo continua: isso foi testado com Jaeger, Prometheus e Loki desligados."

## Bloco 7 — Encerramento (12:40 a 13:10)

**Tela:** README do `tech-challenge-docs` (índice de ADRs e cards).

**Falar:**
> "Os três repositórios, a documentação, os ADRs, a collection do Postman e este vídeo estão no PDF de entrega. Obrigado."

---

## Se algo der errado na gravação

| Situação | O que fazer |
|---|---|
| Um "Aguardar" falha por tempo | Conferir `kubectl -n oficina-mecanica get pods`. Se um pod reiniciou, esperar o `Running` e rodar a pasta de novo: cada execução abre OSs novas |
| Grafana sem dados | O Prometheus lê a cada 15 s e o estado das Sagas a cada 30 s: esperar 30 s |
| Jaeger sem o trace | Conferir se o `correlationId` copiado é do bloco 3 e se o lookback cobre a última hora |
| Passar de 15 min | Cortar a pasta 5 do bloco 4 e o item 5 do bloco 5 |
