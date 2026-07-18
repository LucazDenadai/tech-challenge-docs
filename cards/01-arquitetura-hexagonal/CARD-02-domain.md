# CARD-02 — Domain layer do Atendimento

**Tipo:** Refactor  
**Status:** To Do  
**Depende de:** CARD-01  
**Bloqueia:** CARD-03

---

## Contexto

O Domain é o núcleo da arquitetura hexagonal. Não pode ter dependência de nenhuma biblioteca externa nem conhecer Infrastructure ou Application. Vamos migrar as entidades da Fase 1 limpando o que não pertence a esta camada.

---

## Regra de ouro

> Se precisar de `using` para algo que não é primitivo .NET, está errado.

---

## Critérios de aceite

- [ ] Nenhum `using` externo (sem EF Core, sem Npgsql, sem BCrypt, sem JWT)
- [ ] Nenhuma referência a outros projetos da solution
- [ ] State machine de `OrdemServico` testável sem banco
- [ ] Collections expostas como `IReadOnlyCollection` (sem setter público)
- [ ] `dotnet build` sem warnings

---

## O que migrar do legado

| Arquivo legado | Destino |
|---|---|
| `TechChallenge.Domain/Entities/EntityBase.cs` | `Domain/Entities/EntityBase.cs` |
| `TechChallenge.Domain/Entities/OrdemServico.cs` | `Domain/Entities/OrdemServico.cs` |
| `TechChallenge.Domain/Entities/Cliente.cs` | `Domain/Entities/Cliente.cs` |
| `TechChallenge.Domain/Entities/Veiculo.cs` | `Domain/Entities/Veiculo.cs` |
| `TechChallenge.Domain/Entities/Servico.cs` | `Domain/Entities/Servico.cs` |
| `TechChallenge.Domain/Entities/Peca.cs` | `Domain/Entities/Peca.cs` |
| `TechChallenge.Domain/Entities/ItemServico.cs` | `Domain/Entities/ItemServico.cs` |
| `TechChallenge.Domain/Entities/ItemPeca.cs` | `Domain/Entities/ItemPeca.cs` |
| `TechChallenge.Domain/Entities/HistoricoStatusOS.cs` | `Domain/Entities/HistoricoStatusOS.cs` |
| `TechChallenge.Domain/Enums/StatusOrdemServico.cs` | `Domain/Enums/StatusOrdemServico.cs` |
| `TechChallenge.Domain/Enums/PerfilUsuario.cs` | `Domain/Enums/PerfilUsuario.cs` |
| `TechChallenge.Domain/Validators/CpfValidator.cs` | `Domain/Validators/CpfValidator.cs` |
| `TechChallenge.Domain/Validators/CnpjValidator.cs` | `Domain/Validators/CnpjValidator.cs` |
| `TechChallenge.Domain/Validators/PlacaValidator.cs` | `Domain/Validators/PlacaValidator.cs` |

> As interfaces de repositório que estavam no Domain legado (`IOrdemServicoRepository` etc.) **não vêm para cá** — elas pertencem ao Application (portas de saída). Ver CARD-03.

---

## Passos

1. Copiar entidades e enums do legado
2. Remover qualquer `using` de biblioteca externa
3. Validar que `OrdemServico` encapsula a state machine internamente (sem lógica de transição no service)
4. Validar com `dotnet build`
