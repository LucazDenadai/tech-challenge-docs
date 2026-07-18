# CARD-28 — Infraestrutura AWS via Terraform (EKS, RDS, API Gateway)

**Tipo:** Infra
**Status:** Em andamento — executado em conjunto com CARD-27 (ver nota no CARD-27)
**Depende de:** CARD-26, [ADR-010](../../adr/ADR-010-sizing-e-regiao-aws.md) (região/sizing decididos)
**Bloqueia:** CARD-29, CARD-30
**Decisão arquitetural:** [ADR-009](../../adr/ADR-009-migracao-aws-e-separacao-repositorios.md), [ADR-010](../../adr/ADR-010-sizing-e-regiao-aws.md)

---

## Contexto

Substituir os módulos Kind/Docker (ADR-005) por equivalentes AWS reais, mantendo a mesma separação de responsabilidades entre `tech-challenge-infra-k8s` (cluster + API Gateway) e `tech-challenge-infra-db` (banco gerenciado). Sizing e região já fechados no ADR-010: `us-east-1`, 2 nós `t3.small`, RDS `db.t3.micro`, uso sob demanda (provisionar/destruir por sessão, não 24/7).

Este card é executado junto com o CARD-27 — ver a nota de execução registrada lá. O bootstrap do backend remoto (S3 + DynamoDB) e da IAM role/OIDC é pré-requisito e está descrito no CARD-27.

---

## Critérios de aceite

- [ ] `tech-challenge-infra-k8s`: Terraform provisiona VPC, subnets públicas/privadas, cluster EKS, node group (tamanho mínimo viável, ex: 2 nós `t3.small`)
- [ ] `tech-challenge-infra-k8s`: API Gateway (REST ou HTTP API) provisionado, com rota(s) apontando para o Application Load Balancer do EKS (via Ingress) e rota de auth apontando para a Lambda
- [ ] `tech-challenge-infra-db`: Terraform provisiona instância RDS PostgreSQL (`db.t3.micro`), security group liberando acesso apenas da VPC do EKS
- [ ] Schemas `atendimento` e `estoque` criados no RDS (mesma decisão do ADR-007 — banco compartilhado, schemas separados)
- [ ] Outputs documentados: `cluster_endpoint`, `kubeconfig` (via `aws eks update-kubeconfig`), `rds_connection_string`, `api_gateway_url`
- [ ] `terraform.tfstate` remoto (S3 + DynamoDB lock) — não local, pois múltiplos repositórios/pipelines vão aplicar
- [ ] Nenhuma credencial hardcoded — tudo via variáveis ou Secrets Manager
- [ ] `terraform destroy` funcional nos dois repositórios, documentado no README

---

## Recursos provisionados

### `tech-challenge-infra-k8s`
- VPC + subnets (públicas para ALB, privadas para nós EKS)
- Cluster EKS + node group
- API Gateway (REST/HTTP API)
- Namespaces `oficina-mecanica` e `observabilidade` (via provider kubernetes, como no ADR-005)

### `tech-challenge-infra-db`
- Instância RDS PostgreSQL
- Security group restrito à VPC do EKS
- Schemas `atendimento` e `estoque`

---

## Providers e módulos utilizados

| Provider/Módulo | Função |
|---|---|
| `hashicorp/aws` | RDS, API Gateway, IAM, recursos AWS diretos |
| `terraform-aws-modules/vpc/aws` | VPC, subnets públicas/privadas — módulo oficial da comunidade (ADR-010) |
| `terraform-aws-modules/eks/aws` | Cluster EKS, node group, IAM roles do cluster — módulo oficial da comunidade (ADR-010) |
| `hashicorp/kubernetes` | Namespaces dentro do EKS |

---

## Passos

1. Bootstrap manual do backend remoto (S3 + DynamoDB) e da IAM role/OIDC — ver CARD-27
2. Configurar bloco `backend "s3"` em ambos os repositórios apontando para o bucket/tabela criados
3. `tech-challenge-infra-k8s`: provisionar VPC via módulo `terraform-aws-modules/vpc/aws`
4. `tech-challenge-infra-k8s`: provisionar cluster EKS via módulo `terraform-aws-modules/eks/aws` (2 nós `t3.small`, conforme ADR-010)
5. `tech-challenge-infra-k8s`: provisionar API Gateway com rota de auth (Lambda) e rota proxy (ALB do EKS)
6. `tech-challenge-infra-db`: escrever recursos RDS (`db.t3.micro`) com security group restrito à VPC do EKS
7. `tech-challenge-infra-db`: script de inicialização de schemas (`atendimento`, `estoque`)
8. Criar pipeline CI/CD em ambos os repositórios (`terraform fmt -check`, `plan` em PR, `apply` em merge na `main`) via OIDC — critérios de aceite no CARD-27
9. Validar ciclo `init` → `plan` → `apply` → `destroy` em ambos, confirmando que o custo por hora bate com o estimado no ADR-010
10. Documentar outputs e pré-requisitos nos READMEs, incluindo o lembrete de `terraform destroy` após cada sessão (uso sob demanda)
