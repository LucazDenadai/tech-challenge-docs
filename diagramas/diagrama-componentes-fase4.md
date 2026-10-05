# Diagrama de Componentes — Fase 4

**Status:** Arquitetura aceita nos ADR-014/015. Serviços e recursos ainda não foram implementados.

```mermaid
flowchart LR
    Client[Cliente / Atendente]

    subgraph Platform[Plataforma compartilhada]
        APIGW[API Gateway / Ingress]
        Broker[Broker de mensagens]
        subgraph Cluster[Kubernetes compartilhado]
            OS[Deployment OS\nrepo: tech-challenge-os]
            Billing[Deployment Billing\nrepo: tech-challenge-billing]
            Ops[Deployment Operações\nrepo: tech-challenge-operacoes\nEstoque + Execução]
        end
    end

    Lambda[Lambda de autenticação\ncomponente técnico]

    subgraph Stores[Stores isolados por serviço]
        OSDB[(Banco OS\nCliente, veículo, filial, OS)]
        BillingDB[(Banco Billing\nOrçamento e pagamento)]
        OpsDB[(Banco Operações\nEstoque e execução)]
    end

    Observability[Observabilidade Fase 3]

    Client --> APIGW
    APIGW --> Lambda
    Lambda -->|API interna autenticada\nconsulta cliente| OS
    APIGW --> OS
    APIGW --> Billing
    APIGW --> Ops
    OS <-->|REST quando resposta imediata| Ops
    OS -->|Comandos/eventos correlacionados| Broker
    Billing -->|Eventos de orçamento/pagamento| Broker
    Ops -->|Eventos de reserva/progresso/conclusão| Broker
    Broker --> OS
    Broker --> Billing
    Broker --> Ops
    OS --> OSDB
    Billing --> BillingDB
    Ops --> OpsDB

    OS -.->|traces, métricas, logs| Observability
    Billing -.->|traces, métricas, logs| Observability
    Ops -.->|traces, métricas, logs| Observability
```

## Ownership representado

| Serviço | Fonte da verdade | Não deve fazer |
|---|---|---|
| OS | Clientes, veículos, filiais, ordens, status geral e histórico; associa filial imutável à OS | Escrever em Billing/Operações ou consultar diretamente os bancos deles |
| Billing | Orçamentos, aprovações, pagamentos e referências do Mercado Pago; preserva `filialId` para auditoria/relatórios | Alterar OS/estoque diretamente ou tratar snapshot de cliente como cadastro mestre |
| Operações | Peças, saldos, reservas, movimentações e ciclo de execução; particiona estoque por `(filialId, pecaId)` | Usar saldo de outra filial sem transferência explícita, alterar estado geral da OS ou manter cópia de orçamento/pagamento |

Cluster, rede, API Gateway, broker e observabilidade são plataforma compartilhada, não recursos físicos exclusivos por serviço. Cada serviço possui repositório, banco gerenciado fisicamente dedicado, manifests, permissões e deploy independentes. A Lambda é componente técnico e consulta dados de cliente pela API interna do OS, sem conexão ao banco. Ver [ADR-015](../adr/ADR-015-ownership-e-infraestrutura-fase4.md).