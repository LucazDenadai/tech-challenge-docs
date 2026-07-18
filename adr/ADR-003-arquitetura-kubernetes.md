# ADR-003 — Arquitetura de Deploy no Kubernetes

**Status:** Aceito  
**Data:** 2026-05-27  
**Autores:** Time Tech Challenge — Fase 3

---

## Contexto

Com os dois microserviços implementados (Atendimento e Estoque) e a comunicação assíncrona via RabbitMQ funcionando, o próximo passo é definir como esses serviços serão implantados em Kubernetes. A decisão central é a topologia de Pods: o que roda junto, o que roda separado, como cada componente escala e como o estado persistente é gerenciado.

A arquitetura precisa:
- Permitir escala independente de Atendimento e Estoque (objetivo da Fase 2, descrito no ADR-001)
- Manter o banco de dados e o broker com estado persistente entre reinicializações de Pod
- Ser demonstrável localmente com Kind/Minikube/Docker Desktop K8s
- Usar os mecanismos nativos do Kubernetes (HPA, Probes, Secrets, ConfigMaps) sem dependências externas

---

## Decisão

Cada componente do sistema é implantado como um recurso Kubernetes independente, dentro do namespace `oficina-mecanica`. Nenhum Pod agrupa mais de um serviço de aplicação.

### Topologia do cluster

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                                         CLUSTER K8S                                          │
│                                                                                              │
│  ┌─────────────────────────────┐                                                             │
│  │       CONTROL PLANE         │                                                             │
│  │                             │                                                             │
│  │  ┌──────────────────────┐   │                                                             │
│  │  │  kube-api-server     │◄──┼─────────── kubectl apply -f k8s/                           │
│  │  └──────┬───────────────┘   │                                                             │
│  │         │  ▲                │                                                             │
│  │  ┌──────▼──┐  ┌──────────┐  │                                                             │
│  │  │  etcd   │  │scheduler │  │                                                             │
│  │  └─────────┘  └──────────┘  │                                                             │
│  │  ┌─────────────────────┐    │                                                             │
│  │  │  controller-manager │    │                                                             │
│  │  │  (loop do HPA)      │    │                                                             │
│  │  └─────────────────────┘    │                                                             │
│  └─────────────────────────────┘                                                             │
│                                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────────┐    │
│  │  namespace: oficina-mecanica                                                         │    │
│  │                                                                                      │    │
│  │  ┌──────────────── Node 1 ─────────────────────────────────────────────────────┐    │    │
│  │  │  kubelet + kube-proxy                                                        │    │    │
│  │  │                                                                              │    │    │
│  │  │  ┌─────────────────────────────────┐  ┌───────────────────────────────┐    │    │    │
│  │  │  │  Deployment: atendimento        │  │  Deployment: estoque          │    │    │    │
│  │  │  │  replicas: 2  (HPA: 2→10)       │  │  replicas: 2  (HPA: 2→10)    │    │    │    │
│  │  │  │                                 │  │                               │    │    │    │
│  │  │  │  ┌─────────┐  ┌─────────┐       │  │  ┌─────────┐  ┌─────────┐   │    │    │    │
│  │  │  │  │  pod    │  │  pod    │       │  │  │  pod    │  │  pod    │   │    │    │    │
│  │  │  │  │ :8080   │  │ :8080   │       │  │  │ :8080   │  │ :8080   │   │    │    │    │
│  │  │  │  │ /health │  │ /health │       │  │  │ /health │  │ /health │   │    │    │    │
│  │  │  │  └────┬────┘  └────┬────┘       │  │  └────┬────┘  └────┬────┘   │    │    │    │
│  │  │  └───────┼────────────┼────────────┘  └───────┼────────────┼────────┘    │    │    │
│  │  │          │            │                        │            │             │    │    │
│  │  │  ┌───────▼────────────▼────────────────────────▼────────────▼──────────┐ │    │    │
│  │  │  │  Service (ClusterIP)  atendimento-svc :80 → :8080                    │ │    │    │
│  │  │  │  Service (ClusterIP)  estoque-svc     :80 → :8080                    │ │    │    │
│  │  │  └──────────────────────────────────────────────────────────────────────┘ │    │    │
│  │  │                                                                            │    │    │
│  │  │  ┌──────────────────────────────────────────────────────────────────────┐ │    │    │
│  │  │  │  StatefulSet: rabbitmq              Deployment: postgres             │ │    │    │
│  │  │  │  ┌─────────┐                        ┌─────────┐                      │ │    │    │
│  │  │  │  │  pod    │                        │  pod    │                      │ │    │    │
│  │  │  │  │ :5672   │                        │ :5432   │                      │ │    │    │
│  │  │  │  │ :15672  │                        │ schema: │                      │ │    │    │
│  │  │  │  └────┬────┘                        │ atend.  │                      │ │    │    │
│  │  │  │       │  PVC: rabbitmq-data         │ estoque │                      │ │    │    │
│  │  │  │       └──────────────               └────┬────┘                      │ │    │    │
│  │  │  │                                          │  PVC: postgres-data       │ │    │    │
│  │  │  │  Service (ClusterIP): rabbitmq-svc        └──────────────             │ │    │    │
│  │  │  │  Service (ClusterIP): postgres-svc                                   │ │    │    │
│  │  │  └──────────────────────────────────────────────────────────────────────┘ │    │    │
│  │  └────────────────────────────────────────────────────────────────────────────┘    │    │
│  │                                                                                     │    │
│  │  ┌──────────────── Node 2 (pods extras via HPA) ───────────────────────────┐       │    │
│  │  │  kubelet + kube-proxy                                                    │       │    │
│  │  │                                                                          │       │    │
│  │  │  ┌─────────┐  ┌─────────┐   ← criados automaticamente quando CPU > 70% │       │    │
│  │  │  │  pod    │  │  pod    │                                                │       │    │
│  │  │  │ atend.  │  │ estoque │                                                │       │    │
│  │  │  └─────────┘  └─────────┘                                                │       │    │
│  │  └──────────────────────────────────────────────────────────────────────────┘       │    │
│  │                                                                                      │    │
│  │  ConfigMaps: atendimento-config, estoque-config  (URLs, RabbitMQ host, etc.)        │    │
│  │  Secrets:    atendimento-secrets, estoque-secrets (JWT_KEY, DB_PASSWORD, SMTP)      │    │
│  └──────────────────────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

Fluxo de uma requisição:
  cliente → Service(atendimento-svc) → pod atendimento
                                           │
                                           ├─► postgres-svc → pod postgres (PVC)
                                           │
                                           └─► rabbitmq-svc → pod rabbitmq (PVC)
                                                                    │
                                               Service(estoque-svc) ◄── pod estoque (consome fila)
```

### Tipo de recurso por componente

| Componente | Recurso K8s | Motivo |
|---|---|---|
| Atendimento | `Deployment` | Stateless — pode ter N réplicas idênticas sem coordenação |
| Estoque | `Deployment` | Stateless — idem; consumers MassTransit são seguros para múltiplas réplicas |
| PostgreSQL | `Deployment` + `PVC` | Stateful mas não precisa de identidade de rede estável; PVC garante persistência |
| RabbitMQ | `StatefulSet` + `PVC` | Stateful e requer identidade de rede estável para clustering futuro |

### Por que cada serviço em Pod separado

Um Pod no Kubernetes é a unidade de escalonamento e reinicialização. Colocar Atendimento e Estoque no mesmo Pod:

1. **Inviabiliza escala independente**: o HPA escalaria os dois juntos, mesmo que apenas um esteja sob carga — viola o objetivo central do ADR-001
2. **Acopla ciclos de vida**: uma falha no processo de Estoque derrubaria o Pod inteiro, incluindo o Atendimento
3. **Impede deploys independentes**: atualizar a imagem de um serviço forçaria reinicialização do outro

### Por que banco e broker não escalam horizontalmente

PostgreSQL e RabbitMQ são stateful. Escala horizontal de banco relacional requer soluções específicas (replicação, sharding, Patroni) que estão fora do escopo deste desafio. Ambos ficam com réplica única e PVC — padrão aceito para ambientes de demonstração e pequenas cargas. Em produção real, seriam substituídos por serviços gerenciados (RDS, CloudAMQP).

### Escalonamento automático (HPA)

Apenas Atendimento e Estoque têm HPA configurado, pois são os únicos componentes stateless:

```yaml
minReplicas: 2
maxReplicas: 10
metrics:
  - cpu:    targetAverageUtilization: 70%
  - memory: targetAverageUtilization: 80%
```

Réplica mínima 2 garante disponibilidade mesmo durante rolling updates.

### Probes de saúde

Todos os Pods de aplicação expõem `/health` na porta 8080, implementado via ASP.NET Core Health Checks:

```yaml
readinessProbe:   # Kubernetes só roteia tráfego quando retorna 200
  httpGet: { path: /health, port: 8080 }
  initialDelaySeconds: 10
  periodSeconds: 5

livenessProbe:    # Kubernetes reinicia o Pod se ficar unhealthy
  httpGet: { path: /health, port: 8080 }
  initialDelaySeconds: 30
  periodSeconds: 10
```

### Gestão de configuração e segredos

| Tipo | Conteúdo | Recurso K8s |
|---|---|---|
| Sensível | JWT_KEY, DB_PASSWORD, SMTP_PASSWORD, RABBITMQ_PASSWORD | `Secret` (Opaque) |
| Não sensível | RABBITMQ_HOST, ESTOQUE_SERVICE_URL, ASPNETCORE_ENVIRONMENT | `ConfigMap` |

Valores reais dos Secrets não são commitados — o CI/CD os injeta via `kubectl create secret` com variáveis do repositório (CARD-14).

### Estrutura de arquivos

```
k8s/
├── namespace.yaml
├── postgres/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── pvc.yaml
│   └── secret.yaml
├── rabbitmq/
│   ├── statefulset.yaml
│   ├── service.yaml
│   └── pvc.yaml
├── atendimento/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   └── hpa.yaml
└── estoque/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── secret.yaml
    └── hpa.yaml
```

---

## Alternativas consideradas

### Alternativa 1: Pod único com todos os serviços (sidecar pattern)

Colocar Atendimento, Estoque e banco no mesmo Pod como containers sidecar.

**Prós:**
- Comunicação via localhost — sem latência de rede entre serviços
- Deploy único e simples

**Contras:**
- Escala conjunta obrigatória — inviabiliza o HPA independente que é requisito
- Falha em qualquer container derruba o Pod inteiro
- Banco como sidecar perde dados no restart do Pod sem PVC dedicado
- Nega completamente o modelo de microserviços já construído

**Por que não escolhemos:** contradiz o objetivo central de escala independente definido no ADR-001.

---

### Alternativa 2: Banco como serviço gerenciado externo ao cluster

Usar RDS (AWS) ou Cloud SQL (GCP) em vez de PostgreSQL no cluster.

**Prós:**
- Elimina a preocupação com persistência e backups do banco
- Alta disponibilidade gerenciada pelo provedor

**Contras:**
- Adiciona dependência de cloud externa — o cluster não funciona offline
- Inviabiliza demonstração local com Kind/Minikube sem custos
- Terraform para o banco ficaria no CARD-13, adicionando complexidade desnecessária ao desafio

**Por que não escolhemos:** o ambiente precisa ser autossuficiente para avaliação local. Serviço gerenciado seria a escolha correta em produção real.

---

### Alternativa 3: RabbitMQ como Deployment (sem StatefulSet)

Usar `Deployment` em vez de `StatefulSet` para o RabbitMQ.

**Prós:**
- Manifesto mais simples, mesmo padrão dos serviços de aplicação

**Contras:**
- RabbitMQ em modo cluster requer identidade de rede estável (nome DNS previsível por pod) — o que `StatefulSet` fornece nativamente
- Sem `StatefulSet`, um eventual scale-out do broker quebraria o cluster interno do RabbitMQ

**Por que não escolhemos:** o `StatefulSet` não adiciona complexidade significativa e preserva a capacidade de evoluir para um cluster RabbitMQ multi-nó sem mudar a topologia.

---

## Consequências

### Positivas
- **Escala granular real:** Atendimento e Estoque escalam de forma totalmente independente via HPA
- **Isolamento de falhas:** falha em um Pod de Estoque não afeta os Pods de Atendimento
- **Rolling updates sem downtime:** `replicas: 2` permite substituição gradual de Pods durante deploy
- **Persistência garantida:** PVCs de Postgres e RabbitMQ sobrevivem a reinicializações de Pod
- **Segredos não expostos:** credenciais nunca ficam em ConfigMaps ou no repositório

### Negativas e mitigações

| Risco | Mitigação |
|---|---|
| Banco sem alta disponibilidade (single pod) | Aceito para o desafio; em prod seria RDS Multi-AZ |
| RabbitMQ sem cluster (single pod) | Aceito para o desafio; em prod seria CloudAMQP ou operador RabbitMQ |
| Migrations precisam rodar antes do tráfego | `initContainer` ou lógica de startup no próprio serviço com retry |
| Secrets em base64 no YAML se commitados por engano | `.gitignore` para arquivos `*secret*.yaml` com valores reais; CI/CD injeta via `kubectl create secret` |

---

## Referências

- [Kubernetes — Documentação oficial: Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes — HPA walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Kubernetes — StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes — Configure Liveness and Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [ADR-001 — Arquitetura de Microserviços com Mensageria Assíncrona](ADR-001-arquitetura-microservicos-mensageria.md)
