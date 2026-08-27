# CARD-31 — Observabilidade corporativa (Datadog)

**Tipo:** Observabilidade
**Status:** Em andamento (ADR e Terraform prontos; instalação real depende de janela de reativação da AWS)
**Depende de:** CARD-30
**Bloqueia:** CARD-32
**Decisão arquitetural:** [ADR-012](../../adr/ADR-012-observabilidade-corporativa-datadog.md)

---

## Contexto

A Fase 2 já tem observabilidade própria (Prometheus, Grafana, Loki, Jaeger — ADR-008). A Fase 3 pede integração com **Datadog ou New Relic** especificamente, com dashboards e alertas voltados a métricas de negócio (ordens de serviço), não apenas infraestrutura.

Este card não substitui a stack OTel existente — [ADR-012](../../adr/ADR-012-observabilidade-corporativa-datadog.md) decide manter as duas em paralelo: Datadog cobre o requisito corporativo da Fase 3, a stack OTel local continua servindo depuração sem depender de conta externa.

**Restrição de custo (ver ADR-009 e ADR-012):** a infraestrutura AWS (EKS, RDS) é mantida desligada fora de janelas de demonstração/avaliação, por ter excedido o budget de $5/mês configurado. A instalação do agente Datadog e a geração de dados reais nos dashboards só acontecem durante uma reativação pontual da infra — não há ambiente ligado continuamente para popular os dashboards.

---

## Critérios de aceite

- [x] ADR-012 registrado: escolha do Datadog e decisão de manter a stack OTel em paralelo
- [x] Terraform do agente Datadog (Helm chart, mesmo padrão do ALB Controller), dashboards e alertas escrito e validado (`tech-challenge-infra-k8s/datadog.tf`) — `terraform validate` OK, `apply` pendente de janela de reativação
- [x] Dashboard como código: latência das APIs (Atendimento e Estoque) — `datadog_dashboard.apis`
- [x] Dashboard como código: consumo de recursos do Kubernetes (CPU, memória) por pod — `datadog_dashboard.kubernetes_resources`
- [x] Dashboard como código: healthchecks e uptime dos serviços — `datadog_dashboard.apis`
- [x] Alerta como código: falha no processamento de ordens de serviço (dead letter queue `estoque.baixa_error`) — `datadog_monitor.dlq_baixa_estoque`
- [x] Alerta como código: taxa de erro elevada nas APIs — `datadog_monitor.taxa_erro_apis`
- [ ] Logs estruturados (JSON) com correlação entre requisições (trace ID) chegando ao Datadog via OTLP — validação requer ambiente ativo
- [x] Dashboard como código: volume diário de ordens de serviço — `datadog_dashboard.ordens_servico`
- [x] Dashboard como código: tempo médio de execução por status (Diagnóstico, Execução, Finalização) — `datadog_dashboard.ordens_servico`
- [x] Dashboard como código: erros e falhas nas integrações (chamada HTTP Atendimento→Estoque, consumo RabbitMQ) — `datadog_dashboard.ordens_servico`
- [ ] Screenshots dos dashboards com dados reais, capturados durante a janela de reativação, para a documentação final (CARD-32)

---

## Passos

1. ~~Escrever ADR com a decisão Datadog vs New Relic~~ — concluído, ver ADR-012
2. ~~Definir dashboards e monitors como código Terraform~~ — concluído, ver `tech-challenge-infra-k8s/datadog.tf`
3. Criar conta/workspace Datadog (trial) e gerar `DATADOG_API_KEY`/`DATADOG_APP_KEY`
4. Cadastrar as duas secrets no GitHub do repositório `tech-challenge-infra-k8s`
5. Na próxima janela de reativação da AWS: aplicar o Terraform da infra + o agente Datadog via Helm
6. Validar dashboards e alerta de falha de processamento populados com dados reais
7. Validar logs estruturados chegando com correlação de trace ID
8. Capturar screenshots/link dos dashboards para a documentação final (CARD-32)
9. Encerrar a janela de reativação (`terraform destroy`) ao concluir a captura
