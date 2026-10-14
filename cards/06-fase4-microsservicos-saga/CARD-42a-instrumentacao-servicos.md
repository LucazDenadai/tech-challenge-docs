# CARD-42a — Instrumentação dos serviços: identidade, métricas de negócio e logs ligados ao trace

**Tipo:** Observabilidade / Código
**Status:** Concluído — PRs mergeados nos três serviços (2026-10-13); span do Mercado Pago não verificado ao vivo
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

## Implementação (2026-10-13)

- **Repositórios:** `tech-challenge-os` PR #10 e #11, `tech-challenge-billing` PR #10, #11 e #12, `tech-challenge-operacoes` PR #8 e #9.
- **Identidade:** `service.name` curto (`os`, `billing`, `operacoes`), `service.namespace=oficina-mecanica`, `service.version` (build arg `VERSAO`: `sha-<commit>` no CI, `local` no Kind local) e `deployment.environment`. No Prometheus, aparecem em `target_info`.
- **Traces só no início de um fluxo (achado durante a implementação):** dos últimos 200 traces do OS no Jaeger, 177 eram SQL avulso do laço da outbox e 23 eram probes; nenhum era da Saga. Um amostrador (`ParentBasedSampler` com raiz só para HTTP recebido, mensagem consumida ou publicada) resolve. `/health`, `/ready` e `/metrics` saem do trace, e o SQL de cada comando sai do log `Information`.
- **`TraceId` no log:** já existia. O host do .NET coloca `TraceId` e `SpanId` nos escopos por padrão, então o passo previsto não foi necessário.
- **`CorrelationId` nos logs do processamento (corrigido no PR #11 do OS e de Operações e no #11 do Billing):** o consumidor abria o escopo de log depois do caso de uso, e as transições da Saga saíam sem o id da Saga. Agora o escopo abre antes, com o id das propriedades AMQP. Um teste no OS falha sem a correção.
- **Métricas:**
  - OS: `os_saga_duracao_segundos` (medida no `SaveChanges`, por onde passa toda transição), `os_sagas_por_estado` e `os_saga_idade_etapa_segundos` (leitura periódica do banco), `os_saga_reenvios`.
  - Três serviços: `outbox_publicacoes` e `mensagens_dlq`.
  - Operações: `operacoes_tempo_na_fila_segundos` e `operacoes_duracao_reparo_segundos`. **Mudança do escopo:** sem contadores de execução e reserva em Operações, porque o OS já conta esses eventos por tipo e resultado ao recebê-los (`os_saga_mensagens`).
- **Billing:** instrumentação do `HttpClient` para a chamada ao Mercado Pago.
- **Testes:** amostrador, filtro de probes, duração da Saga (uma vez, com estado final), retrato dos estados, identidade no `/metrics` (três serviços) e tempo na fila e reparo. O PostgreSQL de teste do OS sobe com `max_connections=300`: um banco por teste, cada um com pool próprio, e o serviço novo em segundo plano passou do limite de 100.
- **Cobertura depois da mudança:** OS 93,4% linhas e 86,1% branches; Billing 94,8% e 81,6%; Operações 97,3% e 83,2%. CI verde na `main` ([OS](https://github.com/LucazDenadai/tech-challenge-os/actions/runs/38490645414), [Billing](https://github.com/LucazDenadai/tech-challenge-billing/actions/runs/38490684166), [Operações](https://github.com/LucazDenadai/tech-challenge-operacoes/actions/runs/38490730047)), com deploy no Kind.

### Pendências

- **Span do Mercado Pago não visto ao vivo:** no Kind, o provedor é o simulador, que não faz chamada HTTP, e a criação do Pix no sandbox continua bloqueada por `500` (CARD-39b).

## Critérios de aceite

- [x] Cada serviço reporta `service.name`, `service.version` e `deployment.environment` corretos no trace e nas métricas.
- [x] Cada log JSON tem `TraceId`, e os logs da Saga podem ser filtrados por `CorrelationId`.
- [ ] A chamada ao Mercado Pago aparece como span no trace da Saga, sem corpo, query string ou credenciais. *(Instrumentação configurada; não verificada ao vivo, ver pendências.)*
- [x] As métricas novas existem em `/metrics` e são cobertas por teste: duração e estado das Sagas, reenvios, execuções, reservas, tempo na fila, publicações da outbox e DLQ. *(Execuções e reservas pelo contador do OS.)*
- [x] Os pods têm as anotações de scrape.
- [x] O gate de cobertura (linhas e branches ≥80%) continua verde nos três serviços.

## Passos

1. Identidade do serviço e `TraceId` nos logs (três serviços, `Dockerfile` e CI com a versão).
2. Métricas de mensageria comuns (outbox e DLQ).
3. Métricas da Saga no OS (duração, estados, idade da etapa, reenvios).
4. Métricas de Operações e instrumentação do `HttpClient` no Billing.
5. Anotações de scrape nos manifests.
