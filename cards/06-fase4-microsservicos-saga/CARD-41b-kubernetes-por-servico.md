# CARD-41b — Kubernetes por serviço: manifests, rollout e deploy

**Tipo:** Plataforma / CD
**Status:** To Do — decisões parciais (2026-10-13); restantes no início do card
**Depende de:** CARD-41a
**Bloqueia:** CARD-42, CARD-43
**Repositórios alvo:** `tech-challenge-os`, `tech-challenge-billing`, `tech-challenge-operacoes`; `tech-challenge-infra-k8s` para a plataforma compartilhada
**Decisão arquitetural:** [ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md) (EKS compartilhado, deploy independente), ADR-009/ADR-011 (EKS desligado fora das janelas, OIDC)

---

## Contexto

Cada serviço precisa de Dockerfile, manifests e deploy próprios em Kubernetes. O EKS da Fase 3 (`tech-challenge-infra-k8s`) é a plataforma compartilhada, mas fica destruído fora das janelas de demonstração por custo (ADR-009). Os repositórios da Fase 4 ainda não têm `AWS_ROLE_ARN`.

## Decisões já tomadas (2026-10-13)

- **Validação do deploy (decidido pelo time):** cada pipeline sobe um cluster **Kind** no CI, aplica os manifests do serviço com as dependências mínimas e espera rollout e health. O job de deploy no EKS fica pronto e só roda quando houver `AWS_ROLE_ARN` no repositório e o cluster estiver ligado, na janela de demonstração.
- **Imagem:** a do CARD-41a, tag `sha-<commit>`; o manifest aponta para ela, nunca para `latest`.

## Escopo

- Manifests por serviço (Deployment, Service, ConfigMap, referência a Secret, probes de liveness/readiness, recursos).
- Job de rollout no Kind e job de deploy no EKS por serviço, com `kubectl rollout status` e procedimento de rollback.
- IaC dos bancos de cada serviço e da tabela DynamoDB de Operações com o `GSI1` (`DynamoDb:CriarTabela` desligado na nuvem).
- Travas da outbox para mais de uma réplica: `FOR UPDATE SKIP LOCKED` em Billing e Operações e atualização condicional no DynamoDB de Operações.
- Rotas no API Gateway e Ingress; webhook do Mercado Pago público; simulador do Billing fora da exposição pública.
- Segredos do Mercado Pago e do JWT em Secret.

## Critérios de aceite

- [ ] Cada serviço tem Dockerfile e manifests Kubernetes versionados em seu repositório, com deploy e health check independentes.
- [ ] Merge aprovado implanta somente a unidade de serviço correspondente e aguarda rollout/health (Kind no CI; EKS na janela de demonstração).
- [ ] Se Terraform for escolhido para provisionamento, plan/apply têm aprovação/permissões adequadas e state remoto/locking seguem ADR-011.
- [ ] Secrets vêm de secret store/ambiente seguro e são mascarados nos logs.
- [ ] Falha de rollout impede conclusão do deploy e há procedimento de rollback/redeploy documentado.

## Passos

1. Definir ferramenta de IaC dos bancos e o formato dos manifests (Kustomize ou YAML puro).
2. Manifests por serviço e job de rollout no Kind.
3. Travas da outbox para réplicas.
4. Job de deploy no EKS condicionado a `AWS_ROLE_ARN`, e rotas/Ingress.
5. Deploy real na janela de demonstração e documentação de ligar/desligar.

## Evidências

- Execução do rollout no Kind por serviço.
- Deploy no EKS na janela de demonstração, com prova de que só o serviço alterado foi implantado.
