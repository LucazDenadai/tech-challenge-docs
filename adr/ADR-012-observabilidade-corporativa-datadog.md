# ADR-012 — Observabilidade Corporativa com Datadog

**Status:** Aceito
**Data:** 2026-08-27
**Autores:** Time Tech Challenge — Fase 3

---

## Contexto

A Fase 2 já tem observabilidade própria via OpenTelemetry: Prometheus (métricas), Grafana (dashboards), Loki (logs) e Jaeger (tracing) — decidido no ADR-008. A Fase 3 exige especificamente integração com uma ferramenta de observabilidade **corporativa** (Datadog ou New Relic), com dashboards e alertas voltados a métricas de negócio (ordens de serviço), não apenas infraestrutura.

Este ADR decide entre Datadog e New Relic, e se a stack OTel da Fase 2 é mantida em paralelo ou substituída.

Restrição adicional (ver [ADR-009](ADR-009-migracao-aws-e-separacao-repositorios.md), seção de mitigação de custo): a infraestrutura AWS (EKS, RDS) é mantida **desligada fora de janelas de demonstração/avaliação**, via `terraform destroy`, para controlar custo — o gasto do mês já excedeu o budget de $5 configurado, o que levou à criação de uma trava automática (IAM deny policy via AWS Budget Action) além do teardown completo. Isso significa que o agente de observabilidade, os dashboards e os alertas descritos abaixo são configurados via Terraform/Helm, mas só ficam **ativos e populados com dados reais** durante os períodos em que a infraestrutura é reativada — não continuamente.

---

## Decisão

### Ferramenta: Datadog

Adotar **Datadog** como ferramenta de observabilidade corporativa.

| Critério | Datadog | New Relic |
|---|---|---|
| Free tier | 14 dias trial completo; depois free tier permanente limitado (5 hosts, 1 dia de retenção) | Free tier permanente (100GB/mês, 1 usuário completo) |
| Integração com EKS | Helm chart oficial (`datadog/datadog`), Agent como DaemonSet, coleta automática de métricas de kubelet/cAdvisor | Helm chart oficial equivalente |
| Ingestão de métricas OTel | Suporta OTLP nativamente (não exige reescrever a instrumentação do ADR-008) | Suporta OTLP nativamente |
| Custo de operação neste projeto | Uso pontual (só durante janelas de demonstração), dentro do free tier/trial | Equivalente |

Ambas as ferramentas atendem igualmente ao requisito do desafio para o escopo e duração de uso deste projeto (dashboards + alertas, uso não contínuo). O critério de desempate foi familiaridade do time com o Datadog Agent e a integração mais direta com métricas de Kubernetes (cAdvisor/kubelet) sem configuração adicional.

### Stack OTel da Fase 2: mantida em paralelo

A stack Prometheus/Grafana/Loki/Jaeger (ADR-008) **não é substituída** — é mantida para observabilidade técnica local/desenvolvimento (sem custo, funciona no Kind local sem depender de conta externa). O Datadog Agent consome os mesmos dados via OTLP (a aplicação já exporta OTel, ADR-008), sem exigir uma segunda instrumentação. Datadog cobre especificamente o requisito da Fase 3 (ferramenta corporativa com dashboards de negócio); a stack OTel cobre depuração local e não depende de infraestrutura AWS ativa.

### Escopo dos dashboards e alertas (Datadog)

- Dashboard de latência das APIs (Atendimento e Estoque)
- Dashboard de consumo de recursos do Kubernetes (CPU, memória) por pod/deployment
- Dashboard de healthchecks e uptime dos serviços
- Dashboard de volume diário de ordens de serviço
- Dashboard de tempo médio de execução por status (Diagnóstico, Execução, Finalização)
- Dashboard de erros nas integrações (HTTP Atendimento→Estoque, consumo RabbitMQ)
- Alerta de falha no processamento de ordens de serviço (taxa de erro acima de threshold, ou mensagens acumulando na dead letter queue `estoque.baixa_error`)

### Provisionamento e janela de ativação

O Datadog Agent é instalado via Terraform (`helm_release`) no mesmo padrão do AWS Load Balancer Controller (ver `tech-challenge-infra-k8s/alb-controller.tf`), condicionado à existência do cluster EKS. Dashboards e monitors (alertas) são definidos como código (Terraform provider `DataDog/datadog`) para serem reproduzíveis a cada reativação, em vez de configurados manualmente na UI — evita perda de configuração entre ciclos de destroy/apply.

---

## Alternativas consideradas

### Alternativa 1: New Relic

Ver comparação acima. Rejeitada por preferência de familiaridade do time, não por diferença técnica relevante para este escopo.

### Alternativa 2: Migrar integralmente para Datadog, removendo a stack OTel local

Descartar Prometheus/Grafana/Loki/Jaeger e depender só do Datadog.

**Por que não:** obrigaria qualquer desenvolvimento/depuração local a depender de uma conta Datadog ativa e de conectividade externa, mesmo fora de janelas de demonstração. A stack local não tem custo e já está validada (ADR-008); manter as duas em paralelo, com o Datadog consumindo os mesmos dados via OTLP, não duplica trabalho de instrumentação.

### Alternativa 3: Manter só a stack OTel, sem ferramenta corporativa

Justificar que Prometheus/Grafana/Jaeger/Loki já atendem o requisito de observabilidade.

**Por que não:** o enunciado da Fase 3 pede explicitamente Datadog **ou** New Relic — uma stack self-hosted, ainda que completa, não atende ao requisito literal.

---

## Consequências

### Positivas
- Atende ao requisito obrigatório de observabilidade corporativa da Fase 3
- Reutiliza a instrumentação OTel já existente (ADR-008) — sem reescrever código de aplicação
- Dashboards/monitors como código (Terraform) sobrevivem a ciclos de destroy/apply

### Negativas e mitigações

| Risco | Mitigação |
|---|---|
| Dashboards só têm dados reais durante janelas de reativação da AWS | Screenshots capturados durante a janela de gravação do vídeo de demonstração (CARD-32) documentam o estado funcional para avaliação |
| Free tier/trial do Datadog pode expirar antes da avaliação final | Uso é pontual e de curta duração (só durante reativação); reavaliar se necessário renovar trial antes da entrega |
| Configuração manual na UI do Datadog se perde a cada destroy | Dashboards e monitors definidos via Terraform (provider `DataDog/datadog`), não criados manualmente |

---

## Referências

- [ADR-008 — Observabilidade com OpenTelemetry](ADR-008-observabilidade-opentelemetry.html)
- [ADR-009 — Migração para AWS e Separação em 4 Repositórios](ADR-009-migracao-aws-e-separacao-repositorios.md)
- [CARD-31 — Observabilidade corporativa](../cards/05-fase3-aws/CARD-31-observabilidade-corporativa.md)
- [Datadog Agent Helm Chart](https://docs.datadoghq.com/containers/kubernetes/installation/?tab=helm)
- [Datadog Terraform Provider](https://registry.terraform.io/providers/DataDog/datadog/latest/docs)
