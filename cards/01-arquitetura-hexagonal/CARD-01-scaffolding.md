# CARD-01 — Scaffolding da nova solution monorepo

**Tipo:** Refactor  
**Status:** To Do  
**Bloqueia:** CARD-02, CARD-03, CARD-04, CARD-05, CARD-06, CARD-07

---

## Contexto

A Fase 1 tem uma solution com 4 projetos em camadas flat. Precisamos criar a estrutura nova do zero com os projetos de Atendimento organizados para arquitetura hexagonal, antes de migrar qualquer código.

---

## Critérios de aceite

- [ ] Solution `TechChallenge.slnx` recriada com os novos projetos
- [ ] Referências entre projetos configuradas corretamente:
  - `Domain` ← sem referências externas
  - `Application` → `Domain`
  - `Infrastructure` → `Application` + `Domain`
  - `API` → `Application` + `Infrastructure`
  - `UnitTests` → `Application` + `Domain`
  - `IntegrationTests` → `API` + `Infrastructure`
- [ ] `dotnet build` passa sem erros em todos os projetos
- [ ] Projetos antigos da Fase 1 removidos da solution (arquivos mantidos como referência em `/legacy`)

---

## Projetos a criar

```
src/
└── Atendimento/
    ├── OficinaMecanica.Atendimento.Domain/
    ├── OficinaMecanica.Atendimento.Application/
    ├── OficinaMecanica.Atendimento.Infrastructure/
    └── OficinaMecanica.Atendimento.API/
tests/
└── Atendimento/
    ├── OficinaMecanica.Atendimento.UnitTests/
    └── OficinaMecanica.Atendimento.IntegrationTests/
```

---

## Passos

1. Criar os 6 projetos com `dotnet new`
2. Adicionar todos à solution
3. Configurar as referências entre projetos (`dotnet add reference`)
4. Mover código legado para `/legacy` (não deletar ainda)
5. Validar com `dotnet build`
