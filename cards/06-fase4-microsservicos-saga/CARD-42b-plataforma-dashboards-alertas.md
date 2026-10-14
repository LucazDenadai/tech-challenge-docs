# CARD-42b — Stack de observabilidade no Kind: dashboards, alertas, runbooks e evidências

**Tipo:** Observabilidade / Plataforma
**Status:** Concluído — PR mergeado no `tech-challenge-infra-k8s` e evidências no Kind local (2026-10-13)
**Depende de:** CARD-42a, CARD-41b
**Bloqueia:** CARD-43
**Repositórios alvo:** `tech-challenge-infra-k8s`, `tech-challenge-docs`
**Decisão arquitetural:** [ADR-022](../../adr/ADR-022-observabilidade-fase4-stack-otel-local.md) (stack OTel local no Kind), [ADR-021](../../adr/ADR-021-kubernetes-local-sem-aws-fase4.md) (Kind), [ADR-008](../../adr/ADR-008-observabilidade-opentelemetry.html)

---

## Contexto

A plataforma do Kind (`tech-challenge-infra-k8s/kind/plataforma`) tem RabbitMQ e Jaeger. Prometheus, Grafana, Loki e Promtail existem só como manifests da Fase 2 no repositório antigo (`Tech-challenge/k8s/observabilidade`), apontando para Atendimento e Estoque. Os dashboards e alertas da Fase 3 estão em Terraform do Datadog (`datadog.tf`), que não é usado na Fase 4 (ADR-022).

## Decisões de implementação (2026-10-13)

- **Onde:** `tech-challenge-infra-k8s/kind/observabilidade/`, uma kustomization separada da plataforma. `kind/subir.sh` aplica as duas; com `SEM_OBSERVABILIDADE=1`, só a plataforma. O job `deploy-kind` dos serviços continua aplicando só a plataforma.
- **Componentes:** Prometheus (scrape por anotação de pod, RabbitMQ pela porta 15692 do plugin `rabbitmq_prometheus`, kube-state-metrics), Grafana (fontes de dados Prometheus, Loki e Jaeger provisionadas; link do log para o trace pelo `TraceId`), Loki e Promtail (logs JSON dos pods do namespace `oficina-mecanica`). Os manifests da Fase 2 são a base, com recursos limitados para caber na máquina da demonstração.
- **Portas na máquina local:** Grafana e Prometheus ganham `extraPortMappings` no `cluster.yaml`, como os serviços. O acesso anônimo ao Grafana fica só como leitura; a senha de admin vem do `segredos.sh`.
- **Dashboards (Grafana, JSON versionado):**
  - **Saga da OS:** Sagas por estado, duração por estado final, idade da etapa mais antiga, reenvios, ações manuais por causa, mensagens por resultado e DLQ.
  - **Serviços:** disponibilidade, taxa de requisições, erros e latência p95 por serviço, publicações da outbox, filas do RabbitMQ, pagamentos (chamadas ao Mercado Pago e webhook) e execuções (eventos e tempo na fila).
  O estado do fluxo vem das métricas de negócio, não só de disponibilidade HTTP.
- **Alertas (regras do Prometheus), cada um com link para o runbook:**

  | Alerta | Condição |
  |---|---|
  | `SagaEmAcaoManual` | `os_sagas_por_estado{estado="ManualActionRequired"} > 0` |
  | `PagamentoPendente` | a etapa mais antiga em `AwaitingPayment` passou de 30 minutos (vencimento do Pix, ADR-019) |
  | `SagaParada` | a etapa mais antiga em um estado com prazo passou de 2 vezes o prazo de resposta (CARD-40b) |
  | `DlqCrescendo` | uma fila `*.dlq` ganhou mensagens nos últimos 10 minutos |
  | `DeployIndisponivel` | réplicas indisponíveis em um Deployment por 5 minutos |
  | `ServicoForaDoAr` | o Prometheus não lê o `/metrics` de um serviço por 2 minutos |

- **Runbooks:** `kind/observabilidade/runbooks.md`, uma seção por alerta, com o que olhar (painel, consulta no Loki por `CorrelationId`, trace no Jaeger), a causa provável e a ação (por exemplo, Cancelar ou Retomar como Admin, ADR-020).
- **Validação no CI do `tech-challenge-infra-k8s`:** workflow que roda `promtool check rules`, valida o JSON dos dashboards e sobe o Kind com a plataforma e a observabilidade até tudo ficar pronto.
- **Custo e janela:** zero; a stack existe enquanto o cluster local existir. O card registra a memória usada.

## Implementação (2026-10-13)

- **Repositório:** `tech-challenge-infra-k8s` PR #18. Workflow `Kind` verde no PR e na [`main`](https://github.com/LucazDenadai/tech-challenge-infra-k8s/actions/runs/38496082770).
- **Stack (`kind/observabilidade`):** Prometheus com 6 regras, kube-state-metrics, Loki com Promtail e Grafana (fontes Prometheus, Loki e Jaeger; dashboards **Saga da OS** e **Serviços**). O RabbitMQ expõe a porta 15692 (`rabbitmq_prometheus`). O `subir.sh` sobe tudo e imprime as senhas; `SEM_OBSERVABILIDADE=1` pula a stack.
- **Corrigido no caminho:**
  - **Promtail sem rótulos:** a config herdada da Fase 2 tirava os rótulos de `__path__`, que guarda o padrão com `*`, não o arquivo. Os logs chegavam sem `servico` e `namespace`. Agora usa a descoberta de pods do Kubernetes.
  - **Alerta de DLQ que não via o primeiro envio:** o contador `mensagens_dlq` nasce valendo 1 e `increase()` dá 0. O alerta passou a usar a profundidade da DLQ no RabbitMQ, cuja série existe desde a declaração da fila.
  - **`PagamentoPendente` cobre também `ReconcilingPayment`:** esse estado não tem prazo na Saga, então nenhum outro alerta o cobria.
- **Bug encontrado pelo dashboard (billing PR #12):** depois de um redeploy, 3 Sagas pararam em `CreatingPayment`. O simulador de pagamento começava a numeração sempre no mesmo valor e repetia a referência de pagamentos antigos do banco (`IX_Pagamentos_ReferenciaProvedor`). A numeração agora começa no instante da subida.

## Evidências (Kind local, 2026-10-13)

| Verificação | Resultado |
|---|---|
| Identidade | `target_info` com `service_name` `os`, `billing` e `operacoes`, `service_version=local`, `deployment_environment=Production` |
| Fluxo com dados sintéticos | BDD 5 de 5 contra o cluster; `os_saga_duracao_segundos_count` por estado final: 1 `Completed`, 3 `Cancelled`, 1 `Compensated`; nenhuma Saga em andamento |
| Traces sem ruído | Todos os traces do OS começam em requisição HTTP ou mensagem; nenhum SQL avulso |
| Uma Saga pelo `correlationId` | Loki: 15 linhas (OS 9, Operações 3, Billing 3). Jaeger (tag `correlation_id`): 5 traces, 117 spans, três serviços; o maior com 50 spans |
| `SagaParada` e `SagaEmAcaoManual` | Disparados pela falha real do simulador: "Saga parada em CreatingPayment há 14m 29s"; aos 20 minutos, ação manual com causa `PrazoEsgotado`. Resolvido pelo runbook: Cancelar como Admin, HTTP 204, `ManualActionRequired → ReleasingReservation → Cancelled` com `InventoryReleased` em 265 ms |
| `ServicoForaDoAr` | Operações com 0 réplicas: disparado, com link do runbook |
| `DlqCrescendo` | Mensagem fora do contrato na fila do OS: disparado ("1 mensagem(ns) esperando análise na DLQ os.saga-os.quote-ready.v1.dlq") e resolvido depois do descarte |
| Telemetria fora do ar | Jaeger, Prometheus e Loki com 0 réplicas: BDD 5 de 5 |
| Dados sensíveis | CPF `11144477735` e e-mail `cliente1@cliente.example` do cliente de teste: 0 linhas no Loki e 0 ocorrências em 11.699 spans do Jaeger (o CPF passa na query string de `/os/clientes?busca=`) |
| Custo e janela | Zero: a stack existe enquanto o cluster local existir. Nó do Kind com tudo no ar: cerca de 2,65 GiB de memória |

**Observações:**

- A Saga não cabe em um trace só: cada ação humana (aprovar orçamento, pagamento, mecânico) é uma requisição HTTP nova e abre um trace. Os traces da mesma Saga se encontram pela tag `correlation_id`, o "contexto correlacionável" do Gherkin do CARD-42.
- `PagamentoPendente` (30 minutos) e `DeployIndisponivel` (5 minutos) não foram exercitados; as regras passaram no `promtool` e no workflow.
- O link do log para o trace (campo derivado `TraceId` no Loki) está configurado, mas não foi testado na interface do Grafana.
- Logo depois de um redeploy, o BDD falhou em 3 a 4 cenários de estoque por causa do bug do simulador. Depois da correção, passou.

## Critérios de aceite

- [x] `kind/subir.sh` sobe a stack, e Grafana e Prometheus abrem na máquina local.
- [x] O dashboard da Saga mostra os três serviços e o estado agregado do fluxo com dados sintéticos.
- [x] Um trace do fluxo feliz cobre abertura da OS, orçamento, pagamento (com o webhook simulado), mensagens e execução, com spans identificáveis por serviço. *(Como contexto correlacionável: 5 traces pela tag `correlation_id`, ver observações.)*
- [x] Logs dos três serviços são filtráveis no Loki por `CorrelationId`, com link para o trace. *(Link configurado; não testado na interface.)*
- [x] Uma falha simulada dispara o alerta correspondente, e o runbook descreve a ação.
- [x] Com o Jaeger ou o Prometheus fora do ar, o fluxo continua funcionando (telemetria não bloqueia o negócio).
- [x] Uma busca no Loki e no Jaeger pelo CPF e pelo e-mail do cliente de teste não encontra nada.
- [x] O workflow do `tech-challenge-infra-k8s` valida regras, dashboards e a subida da stack.

## Passos

1. Manifests da stack no `kind/observabilidade` e portas no `cluster.yaml`.
2. Regras de alerta e runbooks.
3. Dashboards provisionados.
4. `subir.sh` e workflow de validação.
5. Demonstração: fluxo feliz, falha simulada, telemetria fora do ar e busca por dados sensíveis; evidências no card.
