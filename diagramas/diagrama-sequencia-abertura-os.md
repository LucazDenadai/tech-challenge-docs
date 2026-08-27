# Diagrama de Sequência — Abertura e Finalização de Ordem de Serviço

Fluxo desde a abertura da OS pelo atendente até a baixa de estoque assíncrona ao finalizar, conforme [ADR-001](../adr/ADR-001-arquitetura-microservicos-mensageria.md) (mensageria entre Atendimento e Estoque) e [ADR-004](../adr/ADR-004-estoque-fonte-verdade-pecas.md) (Estoque como fonte de verdade de peças).

```mermaid
sequenceDiagram
    actor Atendente
    participant Atend as Atendimento
    participant EstoquePort as Estoque (HTTP síncrono)
    participant DB as RDS PostgreSQL\n(schema atendimento)
    participant MQ as RabbitMQ
    participant EstoqueConsumer as Estoque (consumer)
    participant DBEstoque as RDS PostgreSQL\n(schema estoque)
    actor Cliente

    Atendente->>Atend: POST /ordens-servico\n{clienteId, veiculoId, peças}
    Atend->>Atend: valida veículo pertence ao cliente

    alt OS tem peças informadas
        Atend->>EstoquePort: GET disponibilidade das peças
        EstoquePort-->>Atend: disponível / indisponível
        alt peça indisponível
            Atend-->>Atendente: 400 Bad Request
        end
    end

    Atend->>DB: gera número, cria OS (status: Recebida)
    DB-->>Atend: OS criada
    Atend-->>Atendente: 201 { id, numero }

    Note over Atend,DB: Transições de status seguem sequência fixa:\nRecebida → EmDiagnostico → AguardandoAprovacao → EmExecucao → Finalizada → Entregue\n(ou Cancelada, a qualquer momento antes de Entregue)

    Atendente->>Atend: PATCH /ordens-servico/{id}/status\n{novoStatus: EmExecucao}
    Atend->>DB: adiciona itens de serviço/peça,\nregistra HistoricoStatusOS
    Atend-->>Atendente: 200 OK

    Atendente->>Atend: PATCH /ordens-servico/{id}/status\n{novoStatus: Finalizada}
    Atend->>DB: altera status, registra histórico
    Atend->>MQ: publica evento os.finalizada\n{ordemServicoId, itens: [pecaId, quantidade]}
    Atend->>Atend: envia e-mail de atualização\n(best-effort — falha não bloqueia a transição)
    Atend-->>Atendente: 200 OK

    MQ->>EstoqueConsumer: consome os.finalizada
    EstoqueConsumer->>DBEstoque: SubtrairEstoque(quantidade) por peça

    alt falha ao processar (ex: peça inexistente)
        EstoqueConsumer->>MQ: mensagem rejeitada → dead letter queue\nestoque.baixa_error
    else sucesso
        DBEstoque-->>EstoqueConsumer: estoque atualizado
    end

    Note over Cliente,Atend: A qualquer momento, com token JWT\n(ADR-013)

    Cliente->>Atend: GET /ordens-servico/acompanhar/{numero}\nAuthorization: Bearer {token}
    Atend->>DB: consulta status atual da OS
    Atend-->>Cliente: 200 { status, histórico }
```

## Notas

- A verificação de disponibilidade de peças é **síncrona** (chamada HTTP do Atendimento ao Estoque) — acontece na abertura da OS, antes de persistir, para não criar uma OS que não pode ser atendida.
- A baixa efetiva do estoque é **assíncrona** (evento `os.finalizada` via RabbitMQ) — só acontece quando a OS é de fato finalizada, não na abertura. Ver [ADR-001](../adr/ADR-001-arquitetura-microservicos-mensageria.md) para o racional dessa separação.
- Falha no consumo do evento (ex: peça removida do catálogo entre a abertura e a finalização) vai para a dead letter queue `estoque.baixa_error`, monitorada pelo alerta descrito em [ADR-012](../adr/ADR-012-observabilidade-corporativa-datadog.md)/CARD-31.
- O envio de e-mail ao cliente a cada mudança de status é *best-effort*: uma falha no SMTP não impede a transição de status de ser persistida.
- A transição de status é uma máquina de estados linear (`OrdemServico.AlterarStatus`) — não é possível pular etapas; `Cancelada` é a única exceção, permitida a partir de qualquer status exceto `Entregue`.

Ver também [Diagrama de Componentes](diagrama-componentes.md) e [Diagrama de Sequência — Autenticação via CPF](diagrama-sequencia-autenticacao.md).
