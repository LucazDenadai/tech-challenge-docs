# CARD-42b — Stack de observabilidade no Kind: dashboards, alertas, runbooks e evidências

**Tipo:** Observabilidade / Plataforma
**Status:** To Do — escopo e decisões definidos (2026-10-13)
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

## Critérios de aceite

- [ ] `kind/subir.sh` sobe a stack, e Grafana e Prometheus abrem na máquina local.
- [ ] O dashboard da Saga mostra os três serviços e o estado agregado do fluxo com dados sintéticos.
- [ ] Um trace do fluxo feliz cobre abertura da OS, orçamento, pagamento (com o webhook simulado), mensagens e execução, com spans identificáveis por serviço.
- [ ] Logs dos três serviços são filtráveis no Loki por `CorrelationId`, com link para o trace.
- [ ] Uma falha simulada dispara o alerta correspondente, e o runbook descreve a ação.
- [ ] Com o Jaeger ou o Prometheus fora do ar, o fluxo continua funcionando (telemetria não bloqueia o negócio).
- [ ] Uma busca no Loki e no Jaeger pelo CPF e pelo e-mail do cliente de teste não encontra nada.
- [ ] O workflow do `tech-challenge-infra-k8s` valida regras, dashboards e a subida da stack.

## Passos

1. Manifests da stack no `kind/observabilidade` e portas no `cluster.yaml`.
2. Regras de alerta e runbooks.
3. Dashboards provisionados.
4. `subir.sh` e workflow de validação.
5. Demonstração: fluxo feliz, falha simulada, telemetria fora do ar e busca por dados sensíveis; evidências no card.
