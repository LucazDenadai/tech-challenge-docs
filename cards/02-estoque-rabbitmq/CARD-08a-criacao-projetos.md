# CARD-08a — Estoque: criação dos projetos e adição à solution

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-01  
**Bloqueia:** CARD-08b

---

## Contexto

Ponto de partida do microserviço de Estoque. Nenhum projeto existe ainda em `src/Estoque/` nem em `tests/Estoque/`. Todos os 6 projetos precisam ser criados via `dotnet new` e adicionados à solution `TechChallenge.slnx`.

---

## Critérios de aceite

- [ ] 6 projetos criados nas pastas corretas
- [ ] Todos adicionados à solution `TechChallenge.slnx`
- [ ] Referências entre projetos configuradas (Domain ← Application ← Infrastructure ← API)
- [ ] `dotnet build` sem erros (projetos vazios, mas compilando)

---

## Projetos a criar

```
src/Estoque/
├── OficinaMecanica.Estoque.Domain/          (classlib)
├── OficinaMecanica.Estoque.Application/     (classlib)
├── OficinaMecanica.Estoque.Infrastructure/  (classlib)
└── OficinaMecanica.Estoque.API/             (webapi)

tests/Estoque/
├── OficinaMecanica.Estoque.UnitTests/       (xunit)
└── OficinaMecanica.Estoque.IntegrationTests/ (xunit)
```

---

## Referências entre projetos

| Projeto | Referencia |
|---|---|
| Application | Domain |
| Infrastructure | Application |
| API | Application, Infrastructure |
| UnitTests | Application, Domain |
| IntegrationTests | API, Infrastructure |

---

## Passos

1. Criar os 4 projetos de `src/Estoque/` com `dotnet new classlib` / `dotnet new webapi`
2. Criar os 2 projetos de `tests/Estoque/` com `dotnet new xunit`
3. Adicionar todos à solution: `dotnet sln add`
4. Adicionar as referências entre projetos com `dotnet add reference`
5. Validar com `dotnet build`
