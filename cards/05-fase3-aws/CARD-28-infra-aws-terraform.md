# CARD-28 — Infraestrutura AWS via Terraform (EKS, RDS, API Gateway)

**Tipo:** Infra
**Status:** To Do
**Depende de:** CARD-26
**Bloqueia:** CARD-29, CARD-30
**Decisão arquitetural:** [ADR-009](../../adr/ADR-009-migracao-aws-e-separacao-repositorios.md)

---

## Contexto

Substituir os módulos Kind/Docker (ADR-005) por equivalentes AWS reais, mantendo a mesma separação de responsabilidades entre `tech-challenge-infra-k8s` (cluster + API Gateway) e `tech-challenge-infra-db` (banco gerenciado). Provisionamento mínimo viável para o desafio — sem redundância multi-AZ além do necessário para não estourar custo.

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

## Providers utilizados

| Provider | Função |
|---|---|
| `hashicorp/aws` | VPC, EKS, RDS, API Gateway, IAM |
| `hashicorp/kubernetes` | Namespaces dentro do EKS |

---

## Passos

1. Definir backend remoto do Terraform (S3 + DynamoDB) em ambos os repositórios
2. `tech-challenge-infra-k8s`: escrever módulo VPC (ou usar módulo oficial `terraform-aws-modules/vpc`)
3. `tech-challenge-infra-k8s`: escrever módulo EKS (ou usar módulo oficial `terraform-aws-modules/eks`)
4. `tech-challenge-infra-k8s`: provisionar API Gateway com rota de auth (Lambda) e rota proxy (ALB do EKS)
5. `tech-challenge-infra-db`: escrever módulo RDS com security group restrito
6. `tech-challenge-infra-db`: script de inicialização de schemas (`atendimento`, `estoque`)
7. Validar ciclo `init` → `plan` → `apply` → `destroy` em ambos
8. Documentar outputs e pré-requisitos nos READMEs
