# CARD-41b — Kubernetes por serviço: manifests, rollout no Kind e demonstração

**Tipo:** Plataforma / CD
**Status:** Concluído — PRs mergeados no `tech-challenge-infra-k8s` e nos três serviços (2026-10-13); deploy no Kind verde no CI
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

## Implementação (2026-10-13)

- **Repositórios:** `tech-challenge-infra-k8s` PR #16 (plataforma no Kind) e #17; `tech-challenge-os` PR #8 e #9; `tech-challenge-billing` PR #8 e #9; `tech-challenge-operacoes` PR #6 e #7.
- **Plataforma (`tech-challenge-infra-k8s/kind`):** `cluster.yaml` com as portas da demonstração, `plataforma/` (namespace, RabbitMQ, Jaeger), `segredos.sh` (Secrets gerados a cada subida, idempotente) e `subir.sh`/`derrubar.sh`.
- **Serviços (`k8s/`):** Deployment com startup, liveness e readiness, Service NodePort, ConfigMap e PostgreSQL próprio; Operações também com DynamoDB Local. A connection string monta a senha do Secret por `$(DB_SENHA)`.
- **Job `deploy-kind`:** Kind no runner com a plataforma do `tech-challenge-infra-k8s` e só o serviço do repositório; no PR, imagem construída no runner; na `main`, imagem do GHCR `sha-<commit>` carregada no Kind; espera rollout e `/ready`; em falha, publica pods, eventos e logs.
- **Outbox com réplicas:** PostgreSQL com `FOR UPDATE SKIP LOCKED` em transação nos três serviços; DynamoDB de Operações com reserva condicional por item (`ReservadoAte`, 60 s). Teste de duas réplicas em paralelo nos três repositórios; no OS, o mesmo teste sem a trava falhou com 60 publicações para 30 eventos.
- **Terraform da Fase 3:** o workflow tentava a AWS em todo push (`Could not assume role with OIDC`), já falhando desde 2026-08-31. Passou a rodar só por disparo manual (ADR-021).
- **Demonstração local:** `./kind/subir.sh` subiu os três serviços; os 5 cenários BDD passaram contra o cluster, e o Jaeger mostrou traces com os três serviços (trace da Saga com 23 spans).

### Pendências

- **Token de cliente na demonstração:** sem a Lambda (ADR-021), definir no CARD-43.
- **Trace como evidência:** captura no CARD-42.

## Critérios de aceite

- [x] Cada serviço tem Dockerfile e manifests Kubernetes versionados no próprio repositório, com deploy e health check independentes.
- [x] Cada pipeline implanta só o próprio serviço em Kubernetes e aguarda rollout e readiness; falha de rollout falha o pipeline.
- [x] Na `main`, o deployment usa a imagem com tag do commit, não `latest`.
- [x] Segredos não ficam versionados nos manifests.
- [x] A plataforma compartilhada está versionada no `tech-challenge-infra-k8s`, e um script sobe os três serviços no Kind local.
- [x] Procedimento de rollback documentado.
- [x] Outboxes seguras com mais de uma réplica.

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

Registradas em 2026-10-13:

| Evidência | Resultado |
|---|---|
| `deploy-kind` na `main` com imagem do GHCR | [OS](https://github.com/LucazDenadai/tech-challenge-os/actions/runs/38469871024) (`tech-challenge-os:sha-fcdcb20…` baixada e carregada no Kind), [Billing](https://github.com/LucazDenadai/tech-challenge-billing/actions/runs/38469882596), [Operações](https://github.com/LucazDenadai/tech-challenge-operacoes/actions/runs/38469890339): rollout concluído e `/ready` 200 |
| Travas da outbox | Execuções verdes na `main`: [OS](https://github.com/LucazDenadai/tech-challenge-os/actions/runs/38471659466), [Billing](https://github.com/LucazDenadai/tech-challenge-billing/actions/runs/38471675993), [Operações](https://github.com/LucazDenadai/tech-challenge-operacoes/actions/runs/38471693133) |
| Demonstração local | `kind/subir.sh`: três serviços prontos; BDD 5 de 5 contra o cluster; traces dos três serviços no Jaeger |
