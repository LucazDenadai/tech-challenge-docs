# CARD-12 — Kubernetes: manifestos YAML

**Tipo:** Infra  
**Status:** To Do  
**Depende de:** CARD-11  
**Bloqueia:** CARD-14 (CI/CD deploy)

---

## Objetivo

Implantar todos os componentes do sistema em Kubernetes, criando os manifestos YAML que descrevem o estado desejado do cluster. Ao final, `kubectl apply -f k8s/` sobe o ambiente completo e os dois serviços respondem via Postman.

## Por que Kubernetes?

Docker Compose roda vários containers em **uma máquina**. Kubernetes roda, escala e recupera containers em um **cluster de máquinas**:

- O K8s decide **onde** cada container roda (scheduling)
- Se um container morrer, o K8s **reinicia automaticamente** (self-healing)
- Se a CPU subir, o K8s **cria mais réplicas** (HPA)
- Configuração e segredos são **recursos do cluster**, não arquivos locais

---

## Sub-cards

| Sub-card | Arquivo | Entregável |
|---|---|---|
| CARD-12a | [CARD-12a-kind-cluster.md](CARD-12a-kind-cluster.md) | Cluster local com 2 nós em `Ready` |
| CARD-12b | [CARD-12b-namespace.md](CARD-12b-namespace.md) | Namespace `oficina-mecanica` criado |
| CARD-12c | [CARD-12c-postgres.md](CARD-12c-postgres.md) | Pod postgres rodando com PVC |
| CARD-12d | [CARD-12d-rabbitmq.md](CARD-12d-rabbitmq.md) | Pod rabbitmq-0 rodando com PVC |
| CARD-12e | [CARD-12e-atendimento.md](CARD-12e-atendimento.md) | 2 Pods atendimento + HPA ativos |
| CARD-12f | [CARD-12f-estoque.md](CARD-12f-estoque.md) | 2 Pods estoque + HPA ativos |
| CARD-12g | [CARD-12g-validacao.md](CARD-12g-validacao.md) | Fluxo end-to-end via Postman no cluster |

---

## Ordem de execução

```
12a → 12b → 12c → 12d → 12e → 12f → 12g
```

Cada sub-card tem sua própria validação — só avançar quando o entregável do anterior estiver confirmado.

---

## Critérios de aceite

- [ ] `kubectl apply -f k8s/` sobe todos os recursos sem erro
- [ ] Atendimento acessível em `localhost:30080`, Estoque em `localhost:30081`
- [ ] HPA configurado para ambos os serviços
- [ ] Secrets usados para JWT, credenciais de banco e SMTP
- [ ] ConfigMaps para configurações não sensíveis
- [ ] Todos os Pods com `readinessProbe` e `livenessProbe`
- [ ] RabbitMQ com PersistentVolumeClaim
- [ ] Fluxo end-to-end validado via Postman dentro do cluster
