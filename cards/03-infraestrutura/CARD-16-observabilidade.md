# CARD-16 — Observabilidade: OpenTelemetry + Jaeger + Prometheus + Loki + Grafana

**Tipo:** Infra / Aplicação  
**Status:** To Do  
**Depende de:** CARD-13 (namespace `observabilidade` provisionado)  
**Bloqueia:** nenhum  
**Sub-cards:**
- [CARD-16a](CARD-16a-logs-estruturados.md) — Logs estruturados (pré-requisito)
- [CARD-16b](CARD-16b-otel-instrumentacao.md) — Instrumentação OpenTelemetry no código
- [CARD-16c](CARD-16c-infra-k8s.md) — Infraestrutura K8s da stack de observabilidade
- [CARD-16d](CARD-16d-validacao.md) — Validação end-to-end e correlação

---

## Contexto

Com os dois microsserviços rodando no Kubernetes, precisamos de visibilidade sobre o comportamento do sistema em tempo real. Sem observabilidade, falhas silenciosas (mensagens perdidas no RabbitMQ, latência alta, erros intermitentes) são invisíveis até o usuário reportar.

Este card implementa os **três pilares da observabilidade** com a stack padrão de mercado:

| Pilar | Ferramenta | O que responde |
|---|---|---|
| **Traces** | OpenTelemetry + Jaeger | "Onde essa requisição foi lenta ou falhou?" |
| **Métricas** | Prometheus + Grafana | "Como o sistema está se comportando ao longo do tempo?" |
| **Logs** | Loki + Promtail + Grafana | "O que aconteceu neste contexto específico?" |

O Grafana é a **interface unificada** dos três pilares — você parte de uma métrica anômala, clica no TraceId, vai direto ao trace no Jaeger, e de lá correlaciona com os logs no Loki.

---

## Como as peças se conectam

```
Pod (Atendimento / Estoque)
    │
    ├── spans (OTLP/gRPC) ──────────────────► Jaeger
    │                                              └── UI: localhost:30086
    │
    ├── GET /metrics (HTTP) ◄── scrape ──────── Prometheus
    │                                              └── UI: localhost:30090
    │
    └── stdout JSON ──► Promtail ──────────► Loki
                                                └── Grafana (datasource)
                                                        └── UI: localhost:30300
```

O **Promtail** roda como DaemonSet — um agente por nó que lê os logs dos containers diretamente do filesystem do nó (`/var/log/pods/`) e os envia ao Loki. As aplicações não precisam saber que o Loki existe.

---

## Glossário didático

### OpenTelemetry (OTel)
SDK instalado no código da aplicação. Captura automaticamente spans para cada requisição HTTP, query SQL e mensagem de fila — sem instrumentação manual em cada endpoint. Configurado uma vez no `Program.cs`.

### Span e Trace
Um **span** é uma unidade de trabalho com início, fim e metadados (ex: `POST /ordens-servico`, duração 230ms). Um **trace** é a árvore de spans de uma requisição completa — do recebimento no Atendimento até o consumo no Estoque via RabbitMQ.

### Jaeger
Backend de traces distribuídos. Armazena e visualiza traces. Permite ver: "essa requisição levou 230ms — 20ms na API, 180ms no banco, 30ms no RabbitMQ".

### Prometheus
Banco de dados de séries temporais para métricas. Periodicamente "raspa" (scrape) o endpoint `/metrics` de cada serviço e armazena os valores. Responde: "quantas requisições/s agora?", "qual a latência p95?".

### Loki
Backend de logs. Indexa apenas os metadados (labels como `namespace`, `pod`, `app`) e armazena o conteúdo dos logs comprimido. Muito mais leve que o ElasticSearch para logs.

### Promtail
Agente coletor de logs. Roda em cada nó do cluster, lê os logs dos containers do filesystem e os envia ao Loki com labels automáticas do Kubernetes (namespace, pod, container).

### Grafana
Interface de visualização unificada. Conecta em Prometheus (métricas), Loki (logs) e Jaeger (traces) como datasources. Permite correlacionar os três na mesma tela.

---

## Ordem de execução dos sub-cards

```
CARD-16a  →  CARD-16b  →  CARD-16c  →  CARD-16d
(logs)       (OTel)        (K8s)         (validação)
```

Cada sub-card tem seus próprios critérios de aceite e pode ser comitado separadamente.

---

## Estrutura de arquivos ao final

```
src/
├── Atendimento/OficinaMecanica.Atendimento.API/
│   └── Extensions/OpenTelemetryExtensions.cs
└── Estoque/OficinaMecanica.Estoque.API/
    └── Extensions/OpenTelemetryExtensions.cs

k8s/observabilidade/
├── jaeger.yaml
├── prometheus/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml       ← scrape config
├── loki/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml       ← storage config
├── promtail/
│   ├── daemonset.yaml
│   ├── serviceaccount.yaml
│   └── configmap.yaml       ← pipeline de coleta
└── grafana/
    ├── deployment.yaml
    ├── service.yaml
    └── configmap.yaml       ← datasources pré-configurados
```
