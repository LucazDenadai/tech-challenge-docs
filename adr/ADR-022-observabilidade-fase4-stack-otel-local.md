# ADR-022 — Observabilidade da Fase 4 com a stack OpenTelemetry local no Kind

**Status:** Aceito
**Data:** 2026-10-13
**Autores:** Time Tech Challenge — Fase 4
**Relaciona-se a:** [ADR-008](ADR-008-observabilidade-opentelemetry.html) (stack OTel, reaproveitada), [ADR-012](ADR-012-observabilidade-corporativa-datadog.md) (Datadog da Fase 3, não usado na Fase 4), [ADR-021](ADR-021-kubernetes-local-sem-aws-fase4.md) (Kind, sem AWS), [CARD-42](../cards/06-fase4-microsservicos-saga/CARD-42-observabilidade-distribuida.md)

---

## Contexto

O PDF da Fase 4 pede o "uso de ferramentas de monitoramento e observabilidade já implementadas na Fase 3" e, no vídeo, "monitoramento e rastreamento dos fluxos distribuídos" (requisitos R-15 e E-07 da [matriz](../cards/06-fase4-microsservicos-saga/matriz-requisitos.md)).

A Fase 3 teve duas stacks em paralelo (ADR-012):

- **Datadog**, a ferramenta corporativa, instalada no EKS por Terraform (`tech-challenge-infra-k8s/datadog.tf`), com dashboards e monitors como código;
- **stack OpenTelemetry local** (ADR-008): Prometheus, Grafana, Loki com Promtail e Jaeger, mantida para uso local sem custo.

Em 2026-10-13, o time não tem conta AWS (ADR-021) nem acesso garantido à conta Datadog da Fase 3 (trial vencido ou sem acesso). O Datadog não depende da AWS, mas depende de uma conta ativa e de uma chave de API.

Os três serviços da Fase 4 já exportam traces por OTLP para o Jaeger e métricas no formato Prometheus (`/metrics`), e o trace da Saga atravessa os três serviços pela outbox e pelo RabbitMQ (`traceparent` no cabeçalho, CARD-40a).

---

## Decisão

### Ferramenta: a stack OpenTelemetry da Fase 2/3 no Kind

- **Traces:** Jaeger (já na plataforma do Kind, CARD-41b).
- **Métricas e alertas:** Prometheus, com regras de alerta versionadas; lê as métricas dos serviços, do RabbitMQ (plugin `rabbitmq_prometheus`) e do Kubernetes (kube-state-metrics).
- **Logs:** Loki com Promtail lendo os logs JSON dos contêineres.
- **Dashboards:** Grafana com Prometheus, Loki e Jaeger como fontes de dados e dashboards provisionados por arquivo.
- **Onde:** manifests em `tech-challenge-infra-k8s/kind/observabilidade/`, ao lado da plataforma (ownership do ADR-015 e ADR-021). Sobe junto com a demonstração (`kind/subir.sh`). O deploy do CI de cada serviço não sobe essa stack, para não pesar no runner.
- **A instrumentação não muda de padrão:** OpenTelemetry no código, sem SDK de fornecedor. Trocar o backend continua sendo trocar o destino do OTLP e do scrape.

### Datadog fora da Fase 4

O Datadog não é usado na Fase 4. O `datadog.tf` fica no repositório como histórico da Fase 3, junto com o Terraform, que só roda por disparo manual (ADR-021). Se o time recuperar uma conta, o Agent do Datadog pode receber o mesmo OTLP no Kind sem mudar os serviços (alternativa 1).

### Alertas e runbooks

Alertas são regras do Prometheus, visíveis no Prometheus e no Grafana. Não há envio para e-mail ou chat: na demonstração, o alerta disparado e o runbook ligado a ele são a evidência. Cada regra tem um link para o runbook correspondente.

---

## Alternativas consideradas

### Alternativa 1: Datadog no Kind

Agent do Datadog por Helm no Kind, recebendo OTLP, com dashboards e monitors como na Fase 3.

**Por que não:** exige conta ativa. O plano gratuito não inclui APM nem logs, e o trial dura 14 dias, o que não cobre o desenvolvimento e a gravação do vídeo. É a opção mais fiel ao requisito literal da Fase 3. Fica como possibilidade se houver conta.

### Alternativa 2: as duas stacks, com o Datadog só na gravação

Stack OTel como base e o Datadog ligado na janela do vídeo, como o ADR-012 previa.

**Por que não:** duplica dashboards e alertas e ainda depende da conta.

### Alternativa 3: só o Jaeger, que já está no Kind

**Por que não:** o card pede métricas, dashboards, alertas e logs filtráveis por correlation ID, não só traces.

---

## Consequências

### Positivas

- Sem custo e sem conta externa; o mesmo ambiente roda na máquina local para o vídeo.
- Reaproveita a instrumentação existente e a stack que a Fase 3 manteve em paralelo.
- Dashboards, regras de alerta e configuração versionados no `tech-challenge-infra-k8s`.

### Negativas e riscos

| Risco | Mitigação |
|---|---|
| Avaliador associar "Fase 3" ao Datadog | A stack OTel também é da Fase 3 (ADR-012, mantida em paralelo). A escolha e o motivo ficam neste ADR, no README e no vídeo. |
| Alerta sem notificação externa | Para a demonstração basta o alerta disparado e o runbook. Um Alertmanager com destino real pode ser acrescentado depois. |
| Mais memória na máquina da demonstração | Recursos limitados nos manifests; a stack pode ficar fora quando não for necessária (`subir.sh` aceita desligá-la). |
| Dados somem com o cluster | Aceito para demonstração (ADR-021); evidências ficam registradas no CARD-42. |
