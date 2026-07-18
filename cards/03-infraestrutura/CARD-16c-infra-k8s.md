# CARD-16c — Infraestrutura K8s da stack de observabilidade

**Tipo:** Infra  
**Status:** To Do  
**Depende de:** CARD-16b (OTel instrumentado), CARD-13 (namespace `observabilidade` existe)  
**Bloqueia:** CARD-16d  
**Parte de:** [CARD-16](CARD-16-observabilidade.md)

---

## Contexto

Com as aplicações instrumentadas, precisamos subir a stack de coleta e visualização no namespace `observabilidade` do cluster Kind. Todos os componentes rodam como workloads Kubernetes — sem dependência externa.

---

## Componentes e portas de acesso

| Componente | Função | NodePort (acesso pelo host) |
|---|---|---|
| Jaeger | Coleta e visualiza traces | `30086` (UI) / `30317` (OTLP gRPC) |
| Prometheus | Coleta e armazena métricas | `30090` (UI + API) |
| Loki | Armazena logs | interno (só Grafana acessa) |
| Promtail | Coleta logs dos pods | nenhum (DaemonSet, só envia) |
| Grafana | Dashboard unificado | `30300` (UI) |

> As portas 30090 e 30300 já estão mapeadas no cluster Kind (CARD-13). A 30086 e 30317 precisam ser adicionadas ao `infra/modules/cluster/main.tf`.

---

## Critérios de aceite

- [ ] Jaeger rodando e acessível em `http://localhost:30086`
- [ ] Prometheus rodando com scrape de Atendimento, Estoque e Jaeger configurado
- [ ] Loki rodando no namespace `observabilidade`
- [ ] Promtail rodando como DaemonSet coletando logs de todos os pods do namespace `oficina-mecanica`
- [ ] Grafana rodando em `http://localhost:30300` com datasources pré-configurados (Prometheus, Loki, Jaeger)
- [ ] `kubectl apply -f k8s/observabilidade/` aplica tudo sem erro

---

## Estrutura de arquivos

```
k8s/observabilidade/
├── jaeger.yaml
├── prometheus/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── loki/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── promtail/
│   ├── daemonset.yaml
│   ├── serviceaccount.yaml
│   └── configmap.yaml
└── grafana/
    ├── deployment.yaml
    ├── service.yaml
    └── configmap.yaml
```

---

## Detalhamento por componente

### Jaeger (`jaeger.yaml`)
Usar imagem `all-in-one` — coleta, armazena e serve a UI em um único pod. Storage em memória (adequado para demo).

```yaml
image: jaegertracing/all-in-one:1.58
ports:
  - 4317   # OTLP gRPC (recebe traces das aplicações)
  - 16686  # UI
```

### Prometheus (`prometheus/`)
O `configmap.yaml` define os jobs de scrape:
- `atendimento`: raspa `http://atendimento-svc.oficina-mecanica/metrics` a cada 15s
- `estoque`: raspa `http://estoque-svc.oficina-mecanica/metrics` a cada 15s
- `jaeger`: raspa métricas internas do Jaeger

### Loki (`loki/`)
Usar imagem `grafana/loki:3.x` em modo single-binary. Storage no filesystem local (PVC simples para demo).

O `configmap.yaml` configura:
- Retenção: 24h (adequado para demo)
- Storage: filesystem (`/loki/chunks`)

### Promtail (`promtail/`)
DaemonSet — garante um pod por nó do cluster. Monta o diretório de logs do nó (`/var/log/pods`) e envia ao Loki.

O `configmap.yaml` configura o pipeline:
- Filtra logs do namespace `oficina-mecanica`
- Adiciona labels: `namespace`, `pod`, `container`, `app`
- Envia para `http://loki-svc:3100`

O `serviceaccount.yaml` concede permissão de leitura de pods (necessário para descobrir os labels automaticamente).

### Grafana (`grafana/`)
O `configmap.yaml` pré-configura os três datasources via provisioning — sem precisar configurar manualmente pela UI:

```yaml
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus-svc:9090
    isDefault: true
  - name: Loki
    type: loki
    url: http://loki-svc:3100
  - name: Jaeger
    type: jaeger
    url: http://jaeger-svc:16686
```

---

## Atualização necessária no Terraform

Adicionar mapeamentos de porta no `infra/modules/cluster/main.tf` para que o Jaeger seja acessível pelo host:

```hcl
extra_port_mappings {
  container_port = 30086   # Jaeger UI
  host_port      = 30086
}
extra_port_mappings {
  container_port = 30317   # Jaeger OTLP gRPC
  host_port      = 30317
}
```

> Isso requer `terraform destroy` + `terraform apply` para recriar o cluster com as novas portas.

---

## Passos

1. Atualizar `infra/modules/cluster/main.tf` com portas do Jaeger
2. Criar `k8s/observabilidade/jaeger.yaml`
3. Criar `k8s/observabilidade/prometheus/` (3 arquivos)
4. Criar `k8s/observabilidade/loki/` (3 arquivos)
5. Criar `k8s/observabilidade/promtail/` (3 arquivos)
6. Criar `k8s/observabilidade/grafana/` (3 arquivos)
7. Aplicar com `kubectl apply -f k8s/observabilidade/ -n observabilidade`
8. Verificar todos os pods `Running` no namespace `observabilidade`
