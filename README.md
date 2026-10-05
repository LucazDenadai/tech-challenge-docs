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
postman/      ← Collection e environment para testar/demonstrar a API
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
| [ADR-010](adr/ADR-010-sizing-e-regiao-aws.md) | Sizing, custo e região da infraestrutura AWS (Fase 3) |
| [ADR-011](adr/ADR-011-bootstrap-aws-backend-remoto-oidc.md) | Bootstrap AWS: budget, backend remoto Terraform, OIDC (Fase 3) |
| [ADR-012](adr/ADR-012-observabilidade-corporativa-datadog.md) | Observabilidade corporativa com Datadog (Fase 3) |
| [ADR-013](adr/ADR-013-autenticacao-authorize-aspnet-nao-api-gateway.md) | Autorização via `[Authorize]` no Atendimento, não no API Gateway (Fase 3) |
| [ADR-014](adr/ADR-014-limites-microsservicos-fase4.md) | Limites de domínio: OS, Billing e Operações (Estoque + Execução) (Fase 4) |
| [ADR-015](adr/ADR-015-ownership-e-infraestrutura-fase4.md) | Ownership dos dados e infraestrutura por serviço (Fase 4) |
| [ADR-016](adr/ADR-016-bancos-sql-nosql-fase4.md) | Persistência SQL e NoSQL, DynamoDB Local e execução (Fase 4) |
| [ADR-017](adr/ADR-017-saga-orquestrada-os-fase4.md) | Avaliação de Saga para OS — recomendação pendente de confirmação (Fase 4) |
| [ADR-017](adr/ADR-017-saga-orquestrada-os-fase4.md) | Avaliação de Saga para OS — recomendação pendente de confirmação (Fase 4) |
| [ADR-017](adr/ADR-017-saga-orquestrada-os-fase4.md) | Avaliação de Saga para OS — proposta de orquestração pendente de confirmação (Fase 4) |

## RFCs

| RFC | Decisão |
|---|---|
| [RFC-001](rfcs/RFC-001-escolha-da-nuvem.md) | Escolha da nuvem (AWS) |
| [RFC-002](rfcs/RFC-002-escolha-do-banco-de-dados.md) | Escolha do banco de dados gerenciado (RDS PostgreSQL) |
| [RFC-003](rfcs/RFC-003-estrategia-de-autenticacao.md) | Estratégia de autenticação (CPF + JWT via Lambda) |

## Diagramas

| Diagrama | Conteúdo |
|---|---|
| [Componentes](diagramas/diagrama-componentes.md) | Visão de nuvem, APIs, banco e monitoramento (Fase 3) |
| [Sequência — Autenticação via CPF](diagramas/diagrama-sequencia-autenticacao.md) | Fluxo completo: CPF → JWT → consumo de rota protegida |
| [Sequência — Abertura e Finalização de OS](diagramas/diagrama-sequencia-abertura-os.md) | Fluxo completo: abertura → transições de status → baixa de estoque assíncrona |
| [Entidade-Relacionamento](diagramas/diagrama-er.md) | Modelo de dados, schemas `atendimento`/`estoque`, relacionamentos |
| [Componentes — Fase 4](diagramas/diagrama-componentes-fase4.md) | Proposta de OS, Billing e Operações, ownership, dados e plataforma compartilhada |
| [Sequência — Saga da OS Fase 4](diagramas/diagrama-sequencia-saga-fase4.md) | Proposta de fluxo com compensações para avaliação |
| [Estados — Saga da OS Fase 4](diagramas/diagrama-estados-saga-os-fase4.md) | Proposta de estados persistidos e recuperação da Saga |
| [Sequência — Saga da OS Fase 4](diagramas/diagrama-sequencia-saga-fase4.md) | Proposta de fluxo com compensações para avaliação |
| [Estados — Saga da OS Fase 4](diagramas/diagrama-estados-saga-os-fase4.md) | Proposta de estados e recuperação para avaliação |
| [Sequência — Saga da OS Fase 4](diagramas/diagrama-sequencia-saga-fase4.md) | Proposta de fluxo orquestrado, compensações e resultado incerto de pagamento |
| [Estados — Saga da OS Fase 4](diagramas/diagrama-estados-saga-os-fase4.md) | Proposta de estados persistidos e recuperação da Saga |

## Cards

Organizados por épico em [`cards/`](cards/):

- `01-arquitetura-hexagonal/` — scaffolding, domain, application, infrastructure, API do Atendimento
- `02-estoque-rabbitmq/` — microsserviço de Estoque e mensageria assíncrona
- `03-infraestrutura/` — Docker, Kubernetes, Terraform, observabilidade (Fase 2)
- `04-cicd/` — pipeline de CI/CD (Fase 2)
- `05-fase3-aws/` — separação de repositórios, CI/CD multi-repo, infra AWS, Lambda, API Gateway, observabilidade corporativa (Fase 3)
- `06-fase4-microsservicos-saga/` — decomposição em OS, Billing e Operações (Estoque + Execução), bancos por serviço, Saga, Mercado Pago, CI/CD e entregáveis (Fase 4)
