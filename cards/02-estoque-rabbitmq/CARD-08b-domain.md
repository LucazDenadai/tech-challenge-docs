# CARD-08b — Estoque: Domain layer

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-08a  
**Bloqueia:** CARD-08c

---

## Contexto

Núcleo do microserviço de Estoque. Sem dependências externas — apenas primitivos .NET. A entidade `Peca` é análoga à do Atendimento, mas pertence ao contexto de Estoque e adiciona `QuantidadeEstoque`. `MovimentacaoEstoque` é nova.

---

## Regra de ouro

> Se precisar de `using` para algo que não é primitivo .NET, está errado.

---

## Critérios de aceite

- [ ] Nenhum `using` externo (sem EF Core, sem Npgsql, sem pacotes NuGet)
- [ ] Nenhuma referência a outros projetos da solution
- [ ] `Peca` encapsula a lógica de subtração de estoque (sem setter público em `QuantidadeEstoque`)
- [ ] `dotnet build` sem warnings

---

## Entidades

### `Peca`

```csharp
public class Peca
{
    public Guid Id { get; private set; }
    public string Nome { get; private set; }
    public string Descricao { get; private set; }
    public decimal Valor { get; private set; }
    public int QuantidadeEstoque { get; private set; }

    // Método de domínio — encapsula a regra de negócio
    public void SubtrairEstoque(int quantidade)
    {
        if (quantidade > QuantidadeEstoque)
            throw new InvalidOperationException("Quantidade solicitada maior que o estoque disponível.");
        QuantidadeEstoque -= quantidade;
    }
}
```

### `MovimentacaoEstoque`

```csharp
public class MovimentacaoEstoque
{
    public Guid Id { get; private set; }
    public Guid PecaId { get; private set; }
    public TipoMovimentacao Tipo { get; private set; }
    public int Quantidade { get; private set; }
    public string Motivo { get; private set; }
    public Guid? OsId { get; private set; }  // para idempotência no BaixarEstoque
    public DateTime OcorridoEm { get; private set; }
}
```

> `OsId` nullable — preenchido apenas quando a movimentação origina de uma OS finalizada.

---

## Enum

```csharp
public enum TipoMovimentacao
{
    Entrada,
    Saida
}
```

---

## Passos

1. Criar `Domain/Entities/Peca.cs` com construtores e método `SubtrairEstoque`
2. Criar `Domain/Entities/MovimentacaoEstoque.cs` com `OsId` nullable
3. Criar `Domain/Enums/TipoMovimentacao.cs`
4. Validar que nenhum `using` externo foi introduzido
5. `dotnet build` sem warnings
