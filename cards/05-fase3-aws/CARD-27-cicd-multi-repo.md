# CARD-27 — CI/CD nos 4 repositórios (proteção de branch + deploy automático)

**Tipo:** CI/CD
**Status:** To Do
**Depende de:** CARD-26
**Bloqueia:** CARD-28, CARD-29
**Decisão arquitetural:** [ADR-009](../../adr/ADR-009-migracao-aws-e-separacao-repositorios.md), [ADR-006](../../adr/ADR-006-self-hosted-runner-cicd.md) (superseded)

---

## Contexto

Com os repositórios separados (CARD-26), cada um precisa de pipeline próprio. O self-hosted runner do ADR-006 deixa de ser necessário — a AWS é acessível pela internet, então `ubuntu-latest` do GitHub Actions basta. Autenticação via **OIDC** (GitHub Actions ↔ AWS IAM role), sem access keys de longa duração em secrets.

Regras de proteção exigidas pelo desafio: branch `main`/`master` sem commit direto, PR obrigatório para merge, deploy automático para homologação/produção.

---

## Critérios de aceite

- [ ] Branch `main` protegida nos 4 repositórios (`Tech-challenge`, `tech-challenge-lambda`, `tech-challenge-infra-k8s`, `tech-challenge-infra-db`): sem push direto, PR obrigatório, ao menos 1 check obrigatório antes do merge
- [ ] IAM role na AWS configurada para trust do OIDC provider do GitHub Actions (por repositório ou role compartilhada com permissões mínimas)
- [ ] `Tech-challenge`: pipeline builda, testa (`dotnet test` 100%), builda imagem Docker, push para registry, deploy no EKS via `kubectl apply`
- [ ] `tech-challenge-lambda`: pipeline builda, testa, empacota e publica a Lambda (`aws lambda update-function-code` ou SAM/CDK)
- [ ] `tech-challenge-infra-k8s`: pipeline roda `terraform plan` em PR e `terraform apply` no merge em `main`
- [ ] `tech-challenge-infra-db`: pipeline roda `terraform plan` em PR e `terraform apply` no merge em `main`
- [ ] Nenhum secret de access key de longa duração commitado ou salvo como GitHub Secret — apenas ARN da role IAM
- [ ] Jobs sequenciais via `needs:` onde há dependência (build → test → deploy), sem paralelismo sem sentido (lição já registrada no CONTEXT-PROMPT sobre CARD-14)

---

## Passos

1. Criar OIDC provider do GitHub Actions na conta AWS (`token.actions.githubusercontent.com`)
2. Criar IAM role(s) com trust policy restrita a `repo:<org>/<repo>:ref:refs/heads/main`
3. Configurar proteção de branch `main` nos 4 repositórios via `gh api` ou GitHub UI
4. Adaptar pipeline de `Tech-challenge` (baseado no `dotnet.yml` atual) removendo o job `self-hosted` e apontando deploy para EKS
5. Criar pipeline de `tech-challenge-lambda`: build, test, deploy
6. Criar pipeline de `tech-challenge-infra-k8s`: `terraform fmt -check`, `plan` (PR), `apply` (main)
7. Criar pipeline de `tech-challenge-infra-db`: `terraform fmt -check`, `plan` (PR), `apply` (main)
8. Validar um PR de teste em cada repositório para confirmar que o merge dispara o deploy
