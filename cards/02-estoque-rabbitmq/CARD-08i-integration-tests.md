# CARD-08i — Estoque: testes de integração

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-08h  
**Bloqueia:** CARD-09

---

## Contexto

Testes de integração com banco real via Testcontainers, seguindo o mesmo padrão já estabelecido nos testes do Atendimento (`OficinaMecanica.Atendimento.IntegrationTests`). Cobrir os endpoints da API e validar persistência real.

---

## Critérios de aceite

- [ ] `WebApplicationFactory` configurada com Testcontainers (PostgreSQL)
- [ ] Migration aplicada automaticamente antes dos testes
- [ ] Todos os testes listados abaixo passando
- [ ] `dotnet test` no projeto IntegrationTests passa sem erros

---

## Testes obrigatórios

### `PecasControllerTests`

| Teste | Cenário |
|---|---|
| `POST /estoque/pecas — 201` | Cria peça e retorna Id |
| `GET /estoque/pecas — 200` | Lista peças cadastradas |
| `GET /estoque/pecas/{id} — 200` | Retorna peça existente |
| `GET /estoque/pecas/{id} — 404` | Id inexistente |
| `PUT /estoque/pecas/{id} — 204` | Atualiza dados da peça |
| `DELETE /estoque/pecas/{id} — 204` | Remove peça |

### `DisponibilidadeControllerTests`

| Teste | Cenário |
|---|---|
| `POST /estoque/disponibilidade — true` | Todas as peças com estoque suficiente |
| `POST /estoque/disponibilidade — false` | Pelo menos uma peça sem estoque suficiente |

---

## Estrutura de pastas

```
tests/Estoque/OficinaMecanica.Estoque.IntegrationTests/
├── Fixtures/
│   └── EstoqueWebApplicationFactory.cs
├── PecasControllerTests.cs
└── DisponibilidadeControllerTests.cs
```

---

## Referência

Usar `OficinaMecanica.Atendimento.IntegrationTests` como modelo para:
- Configuração do Testcontainers
- `WebApplicationFactory` com override de connection string
- Seed de dados via `DbContext` direto no setup

---

## Passos

1. Instalar `Testcontainers.PostgreSql`, `Microsoft.AspNetCore.Mvc.Testing` no IntegrationTests
2. Criar `EstoqueWebApplicationFactory` baseado no modelo do Atendimento
3. Escrever e implementar `PecasControllerTests` (6 testes)
4. Escrever e implementar `DisponibilidadeControllerTests` (2 testes)
5. `dotnet test` — todos verdes
