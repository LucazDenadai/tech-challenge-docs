# CARD-08g — Estoque: BaixaEstoqueConsumer stub

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-08f  
**Bloqueia:** CARD-08h, CARD-09

---

## Contexto

O consumer RabbitMQ precisa existir como `IHostedService` para que o CARD-09 apenas substitua a implementação interna sem mudar a estrutura. Por ora, apenas loga o evento recebido sem processar — é um stub intencional.

---

## Critérios de aceite

- [ ] `BaixaEstoqueConsumer` registrado como `IHostedService` em `Program.cs`
- [ ] Ao receber uma mensagem, loga o `osId` sem executar baixa
- [ ] Serviço sobe sem erros (mesmo sem RabbitMQ rodando — falha graciosamente)
- [ ] `dotnet build` sem warnings

---

## Implementação esperada

```csharp
// Adapter de entrada — não é controller
public class BaixaEstoqueConsumer : IHostedService
{
    private readonly ILogger<BaixaEstoqueConsumer> _logger;

    public BaixaEstoqueConsumer(ILogger<BaixaEstoqueConsumer> logger)
        => _logger = logger;

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("BaixaEstoqueConsumer iniciado (stub — aguardando CARD-09)");
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
        => Task.CompletedTask;
}
```

---

## Estrutura de pastas

```
API/
└── Adapters/
    └── In/
        └── Messaging/
            └── BaixaEstoqueConsumer.cs
```

---

## Passos

1. Criar `BaixaEstoqueConsumer` como `IHostedService` stub
2. Registrar em `Program.cs`: `builder.Services.AddHostedService<BaixaEstoqueConsumer>()`
3. Validar que o serviço sobe sem RabbitMQ disponível
4. `dotnet build` sem warnings
