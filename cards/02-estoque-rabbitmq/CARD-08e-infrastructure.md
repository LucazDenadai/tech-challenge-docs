# CARD-08e — Estoque: Infrastructure — EF Core + migrations

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-08d  
**Bloqueia:** CARD-08f

---

## Contexto

Implementação dos repositórios e do `AppDbContext` para o schema `estoque`. Padrão idêntico ao do Atendimento (`schema: atendimento`), mas num contexto isolado. As migrations são criadas do zero — o legado não tem schema de estoque.

---

## Critérios de aceite

- [ ] `AppDbContext` configurado com schema `estoque`
- [ ] `PecaRepository` e `MovimentacaoRepository` implementam as interfaces do Application
- [ ] Migration inicial criada e aplicável
- [ ] `dotnet build` sem warnings

---

## AppDbContext

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Peca> Pecas => Set<Peca>();
    public DbSet<MovimentacaoEstoque> Movimentacoes => Set<MovimentacaoEstoque>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasDefaultSchema("estoque");
        // configurações de mapeamento
    }
}
```

---

## Estrutura de pastas

```
Infrastructure/
└── Adapters/
    └── Out/
        └── Persistence/
            ├── AppDbContext.cs
            ├── Repositories/
            │   ├── PecaRepository.cs
            │   └── MovimentacaoRepository.cs
            └── Migrations/
```

---

## Pontos de atenção

- `MovimentacaoRepository.ExisteMovimentacaoPorOsIdAsync` — query por `OsId` nullable, filtrar apenas não-nulos
- Índice único em `MovimentacaoEstoque.OsId` para garantir idempotência a nível de banco
- Connection string via `IConfiguration` / variável de ambiente — sem hardcode

---

## Passos

1. Instalar `Microsoft.EntityFrameworkCore`, `Npgsql.EntityFrameworkCore.PostgreSQL` no Infrastructure
2. Criar `AppDbContext` com schema `estoque`
3. Implementar `PecaRepository`
4. Implementar `MovimentacaoRepository` com query de idempotência
5. Criar migration inicial: `dotnet ef migrations add InitialCreate`
6. `dotnet build` sem warnings
