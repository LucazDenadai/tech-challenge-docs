# CARD-31 — Observabilidade corporativa (Datadog)

**Tipo:** Observabilidade
**Status:** Em andamento (ADR registrado; instalação depende de janela de reativação da AWS)
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
- [ ] Agente Datadog instalado no cluster EKS via Helm chart oficial (Terraform `helm_release`, mesmo padrão do ALB Controller) — durante janela de reativação
- [ ] Dashboard: latência das APIs (Atendimento e Estoque)
- [ ] Dashboard: consumo de recursos do Kubernetes (CPU, memória) por pod/deployment
- [ ] Dashboard: healthchecks e uptime dos serviços
- [ ] Alerta configurado para falhas no processamento de ordens de serviço (ex: taxa de erro > threshold, ou mensagens na dead letter queue `estoque.baixa_error`)
- [ ] Logs estruturados (JSON) com correlação entre requisições (trace ID) chegando ao Datadog via OTLP
- [ ] Dashboard: volume diário de ordens de serviço
- [ ] Dashboard: tempo médio de execução por status (Diagnóstico, Execução, Finalização)
- [ ] Dashboard: erros e falhas nas integrações (chamada HTTP Atendimento→Estoque, consumo RabbitMQ)
- [ ] Screenshots dos dashboards capturados durante a janela de reativação, para a documentação final (CARD-32)

---

## Passos

1. ~~Escrever ADR com a decisão Datadog vs New Relic~~ — concluído, ver ADR-012
2. Criar conta/workspace Datadog (trial)
3. Definir dashboards e monitors como código Terraform (provider `DataDog/datadog`), para sobreviverem a ciclos de destroy/apply
4. Na próxima janela de reativação da AWS: aplicar o Terraform da infra + o agente Datadog via Helm
5. Configurar coleta de métricas de aplicação (latência, taxa de erro) via instrumentação OTel já existente, exportando via OTLP para o Datadog
6. Validar dashboards e alerta de falha de processamento populados com dados reais
7. Validar logs estruturados chegando com correlação de trace ID
8. Capturar screenshots/link dos dashboards para a documentação final (CARD-32)
9. Encerrar a janela de reativação (`terraform destroy`) ao concluir a captura
