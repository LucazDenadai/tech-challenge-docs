# ADR-011 — Bootstrap AWS: backend remoto, OIDC/IAM e proteção de custo

**Status:** Aceito
**Data:** 2026-08-25
**Autores:** Time Tech Challenge — Fase 3

---

## Contexto

O CARD-27 e o CARD-28 exigem, como pré-requisito manual (fora do Terraform, por problema de bootstrap circular — o backend remoto não pode gerenciar a si mesmo), a criação de: backend remoto do Terraform state (S3 + DynamoDB), OIDC provider do GitHub Actions e IAM role de deploy. O ADR-010 já havia decidido região, sizing e o padrão de uso sob demanda, mas não registrava os recursos concretos criados nem a proteção de custo efetivamente configurada. Este ADR registra o que foi executado.

---

## Decisão

### Proteção de custo: AWS Budgets antes de qualquer outro recurso

Antes de criar qualquer recurso de infraestrutura, foi criado um AWS Budget mensal como primeira ação — não depois. Critério: a conta usada não tem free tier (ADR-010), então qualquer recurso esquecido ligado gera cobrança real desde a primeira hora.

- **Budget:** `tech-challenge-monthly`, limite **US$ 5/mês**
- **Alertas por e-mail** (`lucazdenadai@hotmail.com`):
  - 80% do gasto real
  - 100% do gasto real
  - 100% do gasto **previsto** (forecast) — avisa antes de estourar, com base na tendência de consumo, não só depois do fato
- Limite de US$5 é intencionalmente apertado (abaixo da estimativa de ~US$1,50/sessão de 7h do ADR-010): o objetivo é ser avisado cedo em caso de esquecimento de `terraform destroy`, não cobrir uso 24/7.

### Backend remoto do Terraform state

| Recurso | Nome | Configuração |
|---|---|---|
| S3 bucket | `tech-challenga-tfstate` | Versionamento ativado, acesso público bloqueado, criptografia AES256 padrão |
| DynamoDB | `tech-challenge-tfstate-lock` | Partition key `LockID`, billing mode `PAY_PER_REQUEST` (custo irrelevante no volume de uso) |

Ambos em `us-east-1`, conforme região decidida no ADR-010.

### OIDC provider + IAM role compartilhada

- **OIDC provider:** `token.actions.githubusercontent.com`, client ID `sts.amazonaws.com`
- **IAM role:** `tech-challenge-github-actions` (`arn:aws:iam::575225901719:role/tech-challenge-github-actions`)
- **Trust policy:** restrita por `StringLike` ao `sub` do token OIDC, por repositório (sem restringir branch/evento):
  - `repo:LucazDenadai/Tech-challenge:*`
  - `repo:LucazDenadai/tech-challenge-lambda:*`
  - `repo:LucazDenadai/tech-challenge-infra-k8s:*`
  - `repo:LucazDenadai/tech-challenge-infra-db:*`

  A versão inicial restringia a `ref:refs/heads/main`, mas isso quebrou o job de `terraform plan` do CARD-27 rodando em `pull_request` — o GitHub OIDC gera um `sub` diferente (`repo:.../pull/<n>/merge`) para esse evento, e o `AssumeRoleWithWebIdentity` falhava com `Not authorized`. Ajustado para wildcard por repositório em 2026-08-25. Aceitável porque os 4 repositórios são pessoais, sem colaboradores externos ou forks — o risco de um PR malicioso assumir a role é baixo nesse contexto.
- **Uma role compartilhada**, não uma por repositório — escolhida por simplicidade de manutenção (ver alternativas abaixo). O secret `AWS_ROLE_ARN` com o ARN acima foi configurado nos 4 repositórios.

### Escopo de permissões da role

Não foi usada `AdministratorAccess`. A policy inline (`tech-challenge-infra-permissions`) cobre apenas o necessário para os módulos Terraform do CARD-28 e os pipelines do CARD-27:
- `s3:GetObject/PutObject/ListBucket` restrito ao bucket do state
- `dynamodb:GetItem/PutItem/DeleteItem` restrito à tabela de lock
- `eks:*`, `rds:*`, `ec2:*`, `apigateway:*`, `lambda:*`, `logs:*` — amplos dentro do serviço, sem restrição de recurso (necessário porque os módulos `terraform-aws-modules/eks` e `/vpc` criam recursos com nomes/IDs gerados dinamicamente)
- `iam:CreateRole/PassRole/AttachRolePolicy` e afins — necessário porque o módulo EKS cria suas próprias service roles (cluster role, node group role) como parte do `apply`

---

## Alternativas consideradas

### Alternativa 1: Uma IAM role por repositório

**Prós:** mínimo privilégio mais estrito — cada pipeline só pode assumir a role do seu próprio repositório, reduzindo o raio de impacto de um workflow comprometido.

**Por que não (por ora):** mais trabalho de manutenção (4 trust policies, 4 policies de permissão) para um projeto acadêmico de curta duração e uso sob demanda. Decisão pode ser revisitada se o projeto evoluir além do escopo do Tech Challenge.

### Alternativa 2: `AdministratorAccess` na role

**Prós:** mais simples, nunca bloqueia por falta de permissão durante o desenvolvimento do Terraform.

**Por que não:** viola princípio de mínimo privilégio para uma role assumida automaticamente por CI/CD — um workflow malicioso ou um erro de configuração no Terraform teria alcance total sobre a conta AWS.

### Alternativa 3: Access keys de longa duração em vez de OIDC

**Prós:** não exige configurar OIDC provider nem trust policy.

**Por que não:** já descartada no CARD-27 — access keys de longa duração em GitHub Secrets são um risco de vazamento permanente; OIDC gera credenciais temporárias por execução, sem segredo de longa duração armazenado.

---

## Consequências

### Positivas
- Nenhuma credencial de longa duração armazenada nos repositórios — autenticação via OIDC com token de curta duração por execução
- Alerta de custo configurado antes de qualquer recurso cobrável (EKS/RDS) existir, reduzindo risco de fatura inesperada
- Backend remoto com lock evita `apply` concorrente entre pipelines dos diferentes repositórios

### Negativas e mitigações

| Risco | Mitigação |
|---|---|
| Role compartilhada dá aos 4 pipelines o mesmo escopo de permissão | Escopo já restrito por serviço (não `AdministratorAccess`); revisar para roles separadas se o projeto crescer |
| Limite de US$5/mês pode ser insuficiente numa sessão de trabalho mais longa com múltiplos `apply`/`destroy` | Threshold de 80%/100% avisa antes de virar problema; ajustar o limite do budget é uma alteração de baixo custo se necessário |
| Bucket S3 e tabela DynamoDB não são gerenciados pelo Terraform (bootstrap manual) | Documentado neste ADR e no README de cada repo de infra — não tentar importar para o state gerenciado |

---

## Referências

- [ADR-009 — Migração para AWS e Separação em Repositórios](ADR-009-migracao-aws-e-separacao-repositorios.md)
- [ADR-010 — Sizing, custo e região da infraestrutura AWS](ADR-010-sizing-e-regiao-aws.md)
- [CARD-27 — CI/CD nos 4 repositórios](../cards/05-fase3-aws/CARD-27-cicd-multi-repo.md)
- [CARD-28 — Infraestrutura AWS via Terraform](../cards/05-fase3-aws/CARD-28-infra-aws-terraform.md)
