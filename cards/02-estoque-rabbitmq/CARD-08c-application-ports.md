# CARD-08c — Estoque: Application — portas de saída

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-08b  
**Bloqueia:** CARD-08d

---

## Contexto

Define as interfaces (portas de saída) que o Application precisa do mundo externo. Nenhuma implementação aqui — apenas contratos. A Infrastructure vai implementá-las no CARD-08e.

---

## Critérios de aceite

- [ ] Interfaces criadas em `Application/Ports/Out/`
- [ ] Nenhuma referência a Infrastructure ou EF Core
- [ ] `dotnet build` sem warnings

---

## Interfaces

### `IPecaRepository`

```csharp
public interface IPecaRepository
{
    Task<Peca?> ObterPorIdAsync(Guid id);
    Task<IEnumerable<Peca>> ObterTodosAsync();
    Task<IEnumerable<Peca>> ObterPorIdsAsync(IEnumerable<Guid> ids);
    Task AdicionarAsync(Peca peca);
    Task AtualizarAsync(Peca peca);
    Task RemoverAsync(Guid id);
}
```

### `IMovimentacaoRepository`

```csharp
public interface IMovimentacaoRepository
{
    Task AdicionarAsync(MovimentacaoEstoque movimentacao);
    Task<bool> ExisteMovimentacaoPorOsIdAsync(Guid osId);  // para idempotência
}
```

---

## Estrutura de pastas

```
Application/
└── Ports/
    └── Out/
        ├── IPecaRepository.cs
        └── IMovimentacaoRepository.cs
```

---

## Passos

1. Criar pasta `Application/Ports/Out/`
2. Criar `IPecaRepository.cs`
3. Criar `IMovimentacaoRepository.cs` com método de idempotência
4. `dotnet build` sem warnings
