# CARD-42a — Instrumentação dos serviços: identidade, métricas de negócio e logs ligados ao trace

**Tipo:** Observabilidade / Código
**Status:** To Do — escopo e decisões definidos (2026-10-13)
**Depende de:** CARD-40, CARD-41b
**Bloqueia:** CARD-42b, CARD-43
**Repositórios alvo:** `tech-challenge-os`, `tech-challenge-billing`, `tech-challenge-operacoes`
**Decisão arquitetural:** [ADR-022](../../adr/ADR-022-observabilidade-fase4-stack-otel-local.md) (stack OTel local), [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md) (estados da Saga), [ADR-019](../../adr/ADR-019-integracao-mercado-pago-fase4.md) (métricas do pagamento)

---

## Contexto

Ponto de partida, verificado no código em 2026-10-13:

- **Traces:** os três serviços exportam OTLP para o Jaeger (ASP.NET Core e EF Core instrumentados). O `traceparent` é gravado na outbox, vai no cabeçalho da mensagem do RabbitMQ e o consumidor continua o trace. Na demonstração do CARD-41b, um trace da Saga teve 23 spans dos três serviços.
- **Identidade do serviço:** só `service.name` (`OficinaMecanica.OS`, `.Billing`, `.Operacoes`). Faltam versão e ambiente.
- **Logs:** JSON no console com escopos; o `CorrelationId` entra no escopo no middleware HTTP e no consumidor. O log não tem o `TraceId`, então não dá para ir do log ao trace.
- **Métricas (`/metrics`, formato Prometheus):** OS conta mensagens da Saga por tipo e resultado e as Sagas que pararam em ação manual. Billing conta chamadas ao Mercado Pago, notificações do webhook e reconciliações manuais. Operações não tem métrica própria.
- **Chamada ao Mercado Pago:** o Billing usa `HttpClient`, mas sem instrumentação de HTTP de saída; a chamada não aparece no trace.
- **Scrape:** os Deployments não indicam ao Prometheus onde ler as métricas.

## Decisões de implementação (2026-10-13)

- **Identidade (todos):** `service.name` passa a ser curto e igual ao `app` do Kubernetes (`os`, `billing`, `operacoes`), mais `service.namespace=oficina-mecanica`, `service.version` e `deployment.environment`. A versão vem de `Servico:Versao`, preenchida no build da imagem (`--build-arg VERSAO=sha-<commit>` no CI, `local` no Kind local). O ambiente vem de `ASPNETCORE_ENVIRONMENT`.
- **Logs ligados ao trace (todos):** `ActivityTrackingOptions` inclui `TraceId` e `SpanId` no escopo de cada log. Com isso, o Grafana vai do log ao trace no Jaeger e o `CorrelationId` continua filtrando a Saga inteira.
- **Métricas do OS (Saga):**
  - `os_saga_duracao_segundos` (histograma), da abertura ao estado terminal, com rótulo `estado_final` (`Completed`, `Cancelled`, `Compensated`).
  - `os_sagas_por_estado` (gauge, rótulo `estado`) e `os_saga_idade_etapa_segundos` (gauge, a etapa mais antiga em cada estado não terminal). Uma leitura periódica do banco do OS calcula os dois. É o que mostra Saga parada e pagamento pendente sem depender de evento.
  - `os_saga_reenvios` (contador, rótulo `estado`), no reenvio por prazo.
  - Os contadores existentes continuam.
- **Métricas de Operações:** `operacoes_execucoes` (contador, rótulo `evento`: iniciada, concluida, falhou, cancelada, rejeitada), `operacoes_reservas` (contador, rótulo `resultado`) e `operacoes_tempo_na_fila_segundos` (histograma, da entrada na fila ao início do reparo).
- **Billing:** instrumentação do `HttpClient` para a chamada ao Mercado Pago virar span filho do trace da Saga, sem corpo e sem query string. Os contadores do ADR-019 continuam.
- **Mensageria (todos, mesmo nome):** `outbox_publicacoes` (contador, rótulo `resultado`: sucesso, falha) no despachante e `mensagens_dlq` (contador, rótulo `canal`) no consumidor. O rótulo `service` do scrape separa os serviços.
- **Scrape:** anotações `prometheus.io/scrape`, `prometheus.io/port` e `prometheus.io/path` no pod de cada serviço (`k8s/deployment.yaml`).
- **Dados sensíveis:** nenhum atributo novo leva CPF, e-mail, placa, token, assinatura do webhook ou payload. Um teste por serviço verifica que os rótulos das métricas novas só usam valores de enumeração.
- **Sem métrica de tamanho da fila em Operações:** a fila é por filial no DynamoDB, e contar exigiria varrer as filiais periodicamente. O tempo na fila e os contadores de execução mostram acúmulo sem essa varredura.

## Critérios de aceite

- [ ] Cada serviço reporta `service.name`, `service.version` e `deployment.environment` corretos no trace e nas métricas.
- [ ] Cada log JSON tem `TraceId`, e os logs da Saga podem ser filtrados por `CorrelationId`.
- [ ] A chamada ao Mercado Pago aparece como span no trace da Saga, sem corpo, query string ou credenciais.
- [ ] As métricas novas existem em `/metrics` e são cobertas por teste: duração e estado das Sagas, reenvios, execuções, reservas, tempo na fila, publicações da outbox e DLQ.
- [ ] Os pods têm as anotações de scrape.
- [ ] O gate de cobertura (linhas e branches ≥80%) continua verde nos três serviços.

## Passos

1. Identidade do serviço e `TraceId` nos logs (três serviços, `Dockerfile` e CI com a versão).
2. Métricas de mensageria comuns (outbox e DLQ).
3. Métricas da Saga no OS (duração, estados, idade da etapa, reenvios).
4. Métricas de Operações e instrumentação do `HttpClient` no Billing.
5. Anotações de scrape nos manifests.
