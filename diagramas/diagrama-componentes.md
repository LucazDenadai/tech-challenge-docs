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
                    Otel["Coletor OTel /\nagente Datadog·New Relic"]
                end
            end
            RDS["RDS PostgreSQL\nschemas: atendimento, estoque"]
        end

        SecretsManager["Secrets Manager\nchave JWT, connection strings"]
    end

    subgraph Obs["Observabilidade externa"]
        DatadogNR["Datadog / New Relic\n(dashboards, alertas)"]
    end

    Browser -->|"POST /auth/cpf"| APIGW
    Browser -->|"rotas protegidas\n(Bearer JWT)"| APIGW
    APIGW -->|"autoriza"| Lambda
    APIGW -->|"proxy autenticado"| Atendimento
    Lambda -->|"consulta cliente"| RDS
    Lambda -.->|"lê chave JWT"| SecretsManager
    Atendimento -.->|"lê secrets"| SecretsManager
    Atendimento -->|"HTTP síncrono\ndisponibilidade de peças"| Estoque
    Atendimento -->|"evento os.finalizada"| RabbitMQ
    RabbitMQ -->|"consome"| Estoque
    Atendimento --> RDS
    Estoque --> RDS
    Atendimento -.->|"métricas/traces/logs"| Otel
    Estoque -.->|"métricas/traces/logs"| Otel
    Otel -.-> DatadogNR
```

## Componentes

| Componente | Responsabilidade |
|---|---|
| API Gateway | Roteamento e controle de acesso — rotas sensíveis exigem JWT válido |
| Lambda (auth por CPF) | Valida CPF, consulta cliente no RDS, emite JWT |
| Cluster EKS | Executa Atendimento e Estoque com HPA (2-10 réplicas) |
| RDS PostgreSQL | Banco gerenciado, schemas `atendimento` e `estoque` (ADR-007) |
| RabbitMQ | Mensageria assíncrona para baixa de estoque (ADR-001) |
| Secrets Manager | Chave JWT e connection strings, sem credenciais hardcoded |
| Datadog / New Relic | Observabilidade corporativa — dashboards e alertas (CARD-31) |

Ver também [Diagrama de Sequência — Autenticação via CPF](diagrama-sequencia-autenticacao.md).
