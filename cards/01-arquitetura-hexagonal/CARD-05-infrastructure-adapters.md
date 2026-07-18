# CARD-05 — Infrastructure: adapters de saída

**Tipo:** Refactor  
**Status:** To Do  
**Depende de:** CARD-03  
**Bloqueia:** CARD-06, CARD-07

---

## Contexto

Infrastructure implementa as portas de saída definidas no Application (CARD-03). Aqui vivem os repositórios EF Core, o `AppDbContext`, migrations e os stubs das integrações externas que serão substituídos em cards futuros.

---

## Critérios de aceite

- [ ] Todos os repositórios implementam as interfaces do Application
- [ ] `AppDbContext` com todas as configurações Fluent API
- [ ] Migrations geradas e aplicando sem erro
- [ ] Stubs de `IEventPublisher`, `IEstoquePort` e `IEmailPort` implementados (apenas log)
- [ ] `docker-compose up` sobe API + PostgreSQL sem erro
- [ ] `dotnet build` sem warnings

---

## O que implementar

### Repositórios (EF Core + Npgsql)

```
Infrastructure/Adapters/Out/Persistence/
├── AppDbContext.cs
├── Configurations/
│   ├── OrdemServicoConfiguration.cs
│   ├── ClienteConfiguration.cs
│   ├── VeiculoConfiguration.cs
│   ├── ItemServicoConfiguration.cs
│   ├── ItemPecaConfiguration.cs
│   ├── HistoricoStatusOSConfiguration.cs
│   ├── ServicoConfiguration.cs
│   ├── PecaConfiguration.cs
│   └── UsuarioConfiguration.cs
├── Repositories/
│   ├── OrdemServicoRepository.cs
│   ├── ClienteRepository.cs
│   ├── VeiculoRepository.cs
│   ├── UsuarioRepository.cs
│   ├── ServicoRepository.cs
│   └── PecaRepository.cs
└── Migrations/
```

### Stubs de integração

```
Infrastructure/Adapters/Out/Stubs/
├── EventPublisherStub.cs     // loga "Evento publicado: {routingKey}" — substituído no CARD-08
├── EstoqueHttpStub.cs        // sempre retorna disponível: true — substituído no CARD-09
└── EmailStub.cs              // loga "Email enviado para: {destinatario}" — substituído no CARD-10
```

---

## Migrations — estratégia de migração do legado

O projeto legado já possui 3 migrations que devem ser preservadas:

| Migration legada | Ação |
|---|---|
| `20260316235937_InitialCreate` | Copiar para o novo projeto, atualizar namespace |
| `20260405170608_AddHistoricoStatusOS` | Copiar para o novo projeto, atualizar namespace |
| `20260419164120_RenameCpfToDocumento` | Copiar para o novo projeto, atualizar namespace |

**Regra:** não recriar o schema do zero. Copiar e ajustar os namespaces garante que `dotnet ef database update` seja idempotente — funciona em banco novo (aplica tudo) e em banco existente (pula o que já foi aplicado).

Após copiar, gerar uma migration adicional apenas se o novo `AppDbContext` tiver diferenças de schema em relação ao legado (ex: nova coluna, renomeação de tabela).

---

## Passos

1. Criar `AppDbContext` e configurações Fluent API (migrar do legado)
2. Implementar `BaseRepository<T>` genérico
3. Implementar repositórios específicos com queries customizadas
4. Criar os três stubs de integração
5. Copiar as 3 migrations do legado para `Infrastructure/Adapters/Out/Persistence/Migrations/` e atualizar os namespaces
6. Rodar `dotnet ef migrations list` para confirmar que as migrations são reconhecidas
7. Gerar migration adicional com `dotnet ef migrations add` apenas se houver diferença de schema
8. Validar `docker-compose up` e aplicação das migrations na subida
