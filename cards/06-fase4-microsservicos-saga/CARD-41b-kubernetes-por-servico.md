# CARD-41b — Kubernetes por serviço: manifests, rollout no Kind e demonstração

**Tipo:** Plataforma / CD
**Status:** To Do — escopo e decisões definidos (2026-10-13)
**Depende de:** CARD-41a
**Bloqueia:** CARD-42, CARD-43
**Repositórios alvo:** `tech-challenge-os`, `tech-challenge-billing`, `tech-challenge-operacoes`; `tech-challenge-infra-k8s` (plataforma compartilhada)
**Decisão arquitetural:** [ADR-021](../../adr/ADR-021-kubernetes-local-sem-aws-fase4.md) (Kind, sem AWS), [ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md) (deploy independente por serviço), [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md)

---

## Contexto

Cada serviço precisa de manifests e deploy automatizado próprios em Kubernetes (R-14). Sem conta AWS, a plataforma é o Kind: efêmero no CI de cada serviço e local para a demonstração ([ADR-021](../../adr/ADR-021-kubernetes-local-sem-aws-fase4.md)).

## Decisões de implementação (2026-10-13)

- **Plataforma compartilhada (`tech-challenge-infra-k8s`, pasta `kind/`):** namespace `oficina-mecanica`, RabbitMQ e Jaeger em manifests Kustomize; configuração do cluster Kind com as portas da demonstração; script `kind/subir.sh` que cria o cluster, aplica a plataforma e os três serviços a partir dos repositórios lado a lado. O Terraform do EKS fica como histórico da Fase 3.
- **Por serviço (`k8s/` com Kustomize):** `base/` com Deployment (1 réplica, probes em `/<servico>/health` e `/<servico>/ready`, limites de recursos), Service, ConfigMap e o PostgreSQL próprio (Operações também com DynamoDB Local). A senha do banco, a chave JWT e outros segredos vêm de um Secret criado fora do Kustomize (no CI e no script), nunca versionado com valor real.
- **Imagem:** na `main`, o deploy usa a imagem do GHCR com a tag `sha-<commit>` (imutável); no PR, a imagem é construída no runner e carregada no Kind (`kind load`), sem publicar.
- **Job `deploy-kind` no CI de cada serviço:** depois de `build-e-testes`, sobe o Kind, aplica a plataforma do `tech-challenge-infra-k8s` e o serviço, espera `kubectl rollout status` e o readiness. Falha de rollout falha o job e publica eventos e logs do pod. Só o serviço do repositório é implantado.
- **Rollback:** documentado com `kubectl rollout undo` e com reaplicar o overlay apontando para a tag anterior.
- **Travas da outbox para mais de uma réplica:** `FOR UPDATE SKIP LOCKED` na leitura das outboxes PostgreSQL (OS, Billing e Operações) e atualização condicional por item na outbox DynamoDB de Operações. Os manifests usam 1 réplica, mas os serviços ficam prontos para escalar.
- **Fora (ADR-021):** EKS, Terraform dos bancos, API Gateway e Lambda.

## Critérios de aceite

- [ ] Cada serviço tem Dockerfile e manifests Kubernetes versionados no próprio repositório, com deploy e health check independentes.
- [ ] Cada pipeline implanta só o próprio serviço em Kubernetes e aguarda rollout e readiness; falha de rollout falha o pipeline.
- [ ] Na `main`, o deployment usa a imagem com tag do commit, não `latest`.
- [ ] Segredos não ficam versionados nos manifests.
- [ ] A plataforma compartilhada está versionada no `tech-challenge-infra-k8s`, e um script sobe os três serviços no Kind local.
- [ ] Procedimento de rollback documentado.
- [ ] Outboxes seguras com mais de uma réplica.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Deploy independente em Kubernetes

  Cenário: Implantar só o serviço alterado
    Dado que um PR altera somente Operações
    Quando o pipeline de Operações roda
    Então só o Deployment de Operações é aplicado no cluster
    E o job espera o rollout e o readiness

  Cenário: Falhar quando o rollout não fica pronto
    Dado um manifest com probe que nunca responde
    Quando o job de deploy roda
    Então o rollout estoura o prazo e o pipeline falha com os eventos do pod
```

## Passos

1. Plataforma compartilhada no `tech-challenge-infra-k8s` (Kind, RabbitMQ, Jaeger, script).
2. Manifests Kustomize de cada serviço.
3. Job `deploy-kind` em cada pipeline.
4. Travas das outboxes para réplicas.
5. Demonstração local com os três serviços e documentação de rollback.

## Evidências

- Execução do `deploy-kind` por serviço.
- Ambiente local com os três serviços prontos pelo script.
