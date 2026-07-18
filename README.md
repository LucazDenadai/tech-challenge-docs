# Tech Challenge — Documentação Arquitetural

Repositório de documentação centralizada do Tech Challenge (pós-graduação em Software Architecture — FIAP). Reúne Architecture Decision Records, RFCs, diagramas e os cards de execução usados para planejar e rastrear o trabalho nos demais repositórios do projeto.

Este repositório existe porque as decisões documentadas aqui atravessam múltiplos dos repositórios do projeto — centralizá-las evita que um avaliador precise reconstruir o histórico de decisões pulando entre repositórios. Ver [ADR-009](adr/ADR-009-migracao-aws-e-separacao-repositorios.md).

---

## Repositórios do projeto

| Repositório | Conteúdo |
|---|---|
| [Tech-challenge](https://github.com/LucazDenadai/Tech-challenge) | Aplicação principal (microsserviços Atendimento + Estoque) executando em Kubernetes |
| [tech-challenge-lambda](https://github.com/LucazDenadai/tech-challenge-lambda) | Function Serverless de autenticação via CPF |
| [tech-challenge-infra-k8s](https://github.com/LucazDenadai/tech-challenge-infra-k8s) | Terraform: cluster Kubernetes (EKS), VPC, API Gateway |
| [tech-challenge-infra-db](https://github.com/LucazDenadai/tech-challenge-infra-db) | Terraform: banco de dados gerenciado (RDS PostgreSQL) |

---

## Estrutura

```
adr/          ← Architecture Decision Records
cards/        ← Cards de execução (planejamento e rastreio de implementação)
rfcs/         ← Request for Comments (decisões técnicas relevantes)
diagramas/    ← Diagramas de componentes e sequência
```

## ADRs

| ADR | Decisão |
|---|---|
| [ADR-001](adr/ADR-001-arquitetura-microservicos-mensageria.md) | Por que dois microsserviços em vez de monolito modular |
| [ADR-002](adr/ADR-002-observabilidade-falhas-tabela-banco.md) | Rastreamento de falhas em tabela de banco em vez de log externo |
| [ADR-003](adr/ADR-003-arquitetura-kubernetes.md) | Estratégia de deploy no Kubernetes (namespaces, HPA, secrets) |
| [ADR-004](adr/ADR-004-estoque-fonte-verdade-pecas.md) | Estoque como fonte de verdade para disponibilidade de peças |
| [ADR-005](adr/ADR-005-infraestrutura-como-codigo-terraform.md) | Kind local via Terraform (superseded pelo ADR-009) |
| [ADR-006](adr/ADR-006-self-hosted-runner-cicd.md) | Self-hosted runner para o deploy (superseded pelo ADR-009) |
| [ADR-007](adr/ADR-007-banco-compartilhado-schemas-separados.md) | Banco compartilhado com schemas separados por serviço |
| [ADR-008](adr/ADR-008-observabilidade-opentelemetry.html) | Observabilidade com OpenTelemetry |
| [ADR-009](adr/ADR-009-migracao-aws-e-separacao-repositorios.md) | Migração para AWS e separação em repositórios (Fase 3) |

## Cards

Organizados por épico em [`cards/`](cards/):

- `01-arquitetura-hexagonal/` — scaffolding, domain, application, infrastructure, API do Atendimento
- `02-estoque-rabbitmq/` — microsserviço de Estoque e mensageria assíncrona
- `03-infraestrutura/` — Docker, Kubernetes, Terraform, observabilidade (Fase 2)
- `04-cicd/` — pipeline de CI/CD (Fase 2)
- `05-fase3-aws/` — separação de repositórios, CI/CD multi-repo, infra AWS, Lambda, API Gateway, observabilidade corporativa (Fase 3)
