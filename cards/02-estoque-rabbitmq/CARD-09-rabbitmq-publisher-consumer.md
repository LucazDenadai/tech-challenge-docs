# CARD-09 — RabbitMQ: publisher no Atendimento + consumer no Estoque

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-05, CARD-08  
**Bloqueia:** CARD-10

---

## Contexto

Substituir os stubs de `IEventPublisher` (Atendimento) e `BaixaEstoqueConsumer` (Estoque) pelas implementações reais com RabbitMQ via MassTransit. Quando uma OS é finalizada, o Atendimento publica o evento `os.finalizada` e o Estoque consome e processa a baixa.

---

## Critérios de aceite

- [ ] RabbitMQ rodando no `docker-compose` (imagem `rabbitmq:3-management`)
- [ ] Atendimento publica evento ao finalizar uma OS
- [ ] Estoque consome o evento e executa `BaixarEstoqueUseCase`
- [ ] Retry automático: falha no consumer recoloca na fila (3 tentativas, dead letter queue)
- [ ] Idempotência: reprocessar o mesmo evento não gera dupla baixa
- [ ] Management UI do RabbitMQ acessível em `localhost:15672` (user: guest/guest)
- [ ] Teste de integração end-to-end validando o fluxo completo

---

## Biblioteca escolhida

**MassTransit** com transport RabbitMQ.

Por que MassTransit e não o client nativo do RabbitMQ:
- Abstrai retry, dead letter queue e idempotência
- Configuração de consumer é declarativa
- Facilita trocar o transport no futuro (ex: SQS) sem mudar os use cases

---

## Contrato do evento

```csharp
// Pacote compartilhado ou namespace comum
public record OsFinalizadaEvent
{
    public Guid EventId { get; init; } = Guid.NewGuid();
    public string EventType => "os.finalizada";
    public DateTimeOffset OcorridoEm { get; init; } = DateTimeOffset.UtcNow;
    public Guid OrdemServicoId { get; init; }
    public string NumeroOS { get; init; } = string.Empty;
    public IReadOnlyList<ItemBaixaDto> Itens { get; init; } = [];
}

public record ItemBaixaDto(Guid PecaId, int Quantidade);
```

---

## Implementação no Atendimento

Substituir `EventPublisherStub` por `RabbitMqEventPublisher`:

```
Infrastructure/Adapters/Out/Messaging/
└── RabbitMqEventPublisher.cs    // implementa IEventPublisher via MassTransit IPublishEndpoint
```

Configuração no `Program.cs`:
```csharp
builder.Services.AddMassTransit(x =>
{
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("rabbitmq", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
    });
});
```

---

## Implementação no Estoque

Substituir `BaixaEstoqueConsumer` stub pela implementação real:

```
Infrastructure/Adapters/In/Messaging/
└── BaixaEstoqueConsumer.cs    // IConsumer<OsFinalizadaEvent>
```

```csharp
public class BaixaEstoqueConsumer : IConsumer<OsFinalizadaEvent>
{
    public async Task Consume(ConsumeContext<OsFinalizadaEvent> context)
    {
        await _baixarEstoqueUseCase.ExecuteAsync(context.Message);
    }
}
```

Configuração com retry:
```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<BaixaEstoqueConsumer>();
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.ReceiveEndpoint("estoque.baixa", e =>
        {
            e.UseMessageRetry(r => r.Intervals(1000, 5000, 10000));
            e.ConfigureConsumer<BaixaEstoqueConsumer>(ctx);
            e.DeadLetterQueueName = "estoque.baixa.dlq";
        });
    });
});
```

---

## docker-compose — adicionar RabbitMQ

```yaml
rabbitmq:
  image: rabbitmq:3-management
  ports:
    - "5672:5672"
    - "15672:15672"
  environment:
    RABBITMQ_DEFAULT_USER: guest
    RABBITMQ_DEFAULT_PASS: guest
  healthcheck:
    test: ["CMD", "rabbitmq-diagnostics", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5
```

---

## Passos

1. Adicionar RabbitMQ ao `docker-compose.yml`
2. Instalar MassTransit nos projetos de Atendimento e Estoque
3. Implementar `RabbitMqEventPublisher` no Atendimento
4. Implementar `BaixaEstoqueConsumer` real no Estoque
5. Configurar retry e dead letter queue
6. Escrever teste de integração end-to-end:
   - Criar e finalizar OS via API do Atendimento
   - Aguardar processamento do consumer
   - Verificar que estoque foi decrementado via API do Estoque
