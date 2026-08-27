# Diagrama de Componentes — Fase 3

Visão de nuvem, APIs, banco de dados e monitoramento após a migração para AWS ([ADR-009](../adr/ADR-009-migracao-aws-e-separacao-repositorios.md)).

```mermaid
flowchart TB
    subgraph Cliente["Cliente / Atendente"]
        Browser["Browser / Postman / App"]
    end

    subgraph AWS["AWS — us-east-1"]
        subgraph Edge["Borda"]
            APIGW["API Gateway"]
            Lambda["Lambda .NET\nauth por CPF"]
        end

        subgraph VPC["VPC"]
            subgraph EKS["Cluster EKS"]
                subgraph NsApp["namespace: oficina-mecanica"]
                    Atendimento["Pod Atendimento\n(HPA 2-10 réplicas)"]
                    Estoque["Pod Estoque\n(HPA 2-10 réplicas)"]
                    RabbitMQ["RabbitMQ\n(StatefulSet)"]
                end
                subgraph NsObs["namespace: observabilidade"]
                    Otel["Coletor OTel /\nagente Datadog"]
                end
            end
            RDS["RDS PostgreSQL\nschemas: atendimento, estoque"]
        end
    end

    subgraph Obs["Observabilidade externa"]
        DatadogExt["Datadog\n(dashboards, alertas — ADR-012)"]
    end

    Browser -->|"POST /atendimento/auth/cpf"| APIGW
    Browser -->|"rotas protegidas\n(Bearer JWT)"| APIGW
    APIGW -->|"invoca (proxy AWS_PROXY)"| Lambda
    APIGW -->|"proxy genérico\n(sem authorizer — ADR-013)"| Atendimento
    Lambda -->|"consulta cliente"| RDS
    Atendimento -.->|"valida JWT via [Authorize]\n(chave HMAC em env var/K8s Secret)"| Atendimento
    Atendimento -->|"HTTP síncrono\ndisponibilidade de peças"| Estoque
    Atendimento -->|"evento os.finalizada"| RabbitMQ
    RabbitMQ -->|"consome"| Estoque
    Atendimento --> RDS
    Estoque --> RDS
    Atendimento -.->|"métricas/traces/logs"| Otel
    Estoque -.->|"métricas/traces/logs"| Otel
    Otel -.-> DatadogExt
```

## Componentes

| Componente | Responsabilidade |
|---|---|
| API Gateway | Proxy genérico para o ALB do EKS + rota de autenticação para a Lambda (sem authorizer na borda — [ADR-013](../adr/ADR-013-autenticacao-authorize-aspnet-nao-api-gateway.md)) |
| Lambda (auth por CPF) | Valida CPF, consulta cliente no RDS, emite JWT (HMAC-SHA256) |
| Cluster EKS | Executa Atendimento e Estoque com HPA (2-10 réplicas) |
| RDS PostgreSQL | Banco gerenciado, schemas `atendimento` e `estoque` (ADR-007) |
| RabbitMQ | Mensageria assíncrona para baixa de estoque (ADR-001) |
| Atendimento (`[Authorize]`) | Valida o JWT nas rotas sensíveis, incluindo `/ordens-servico/acompanhar/{numero}` — [ADR-013](../adr/ADR-013-autenticacao-authorize-aspnet-nao-api-gateway.md) |
| Datadog | Observabilidade corporativa — dashboards e alertas ([ADR-012](../adr/ADR-012-observabilidade-corporativa-datadog.md), CARD-31) |

Ver também [Diagrama de Sequência — Autenticação via CPF](diagrama-sequencia-autenticacao.md).
