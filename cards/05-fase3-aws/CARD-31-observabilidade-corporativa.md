# CARD-31 — Observabilidade corporativa (Datadog/New Relic)

**Tipo:** Observabilidade
**Status:** To Do
**Depende de:** CARD-30
**Bloqueia:** CARD-32
**Decisão arquitetural:** a registrar (ADR-010, ver passo 1)

---

## Contexto

A Fase 2 já tem observabilidade própria (Prometheus, Grafana, Loki, Jaeger — ADR-008). A Fase 3 pede integração com **Datadog ou New Relic** especificamente, com dashboards e alertas voltados a métricas de negócio (ordens de serviço), não apenas infraestrutura.

Este card não substitui a stack OTel existente — decide se ela é mantida em paralelo (métricas técnicas) com Datadog/New Relic cobrindo os requisitos específicos do desafio (dashboards de negócio, alertas), ou se é migrada integralmente. Essa decisão é um tradeoff arquitetural e deve virar ADR antes de implementar.

---

## Critérios de aceite

- [ ] ADR-010 registrado: escolha entre Datadog e New Relic, e decisão sobre manter ou substituir a stack OTel da Fase 2
- [ ] Agente/integração instalado no cluster EKS (DaemonSet ou sidecar, conforme a ferramenta escolhida)
- [ ] Dashboard: latência das APIs (Atendimento e Estoque)
- [ ] Dashboard: consumo de recursos do Kubernetes (CPU, memória) por pod/deployment
- [ ] Dashboard: healthchecks e uptime dos serviços
- [ ] Alerta configurado para falhas no processamento de ordens de serviço (ex: taxa de erro > threshold, ou mensagens na dead letter queue `estoque.baixa_error`)
- [ ] Logs estruturados (JSON) com correlação entre requisições (trace ID) chegando à ferramenta escolhida
- [ ] Dashboard: volume diário de ordens de serviço
- [ ] Dashboard: tempo médio de execução por status (Diagnóstico, Execução, Finalização)
- [ ] Dashboard: erros e falhas nas integrações (chamada HTTP Atendimento→Estoque, consumo RabbitMQ)

---

## Passos

1. Escrever ADR-010 com a decisão Datadog vs New Relic e o tradeoff de manter/substituir OTel
2. Criar conta/workspace na ferramenta escolhida
3. Instalar agente no cluster EKS via Helm chart oficial
4. Configurar coleta de métricas de aplicação (latência, taxa de erro) via instrumentação já existente (OTel, se mantido) ou SDK nativo da ferramenta
5. Configurar dashboards de infraestrutura (CPU, memória, uptime)
6. Configurar dashboards de negócio (volume de OS, tempo por status, erros de integração)
7. Configurar alerta de falha de processamento
8. Validar logs estruturados chegando com correlação de trace ID
9. Capturar screenshots/link dos dashboards para a documentação final (CARD-32)
