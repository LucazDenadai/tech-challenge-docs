# CARD-08 — Microserviço de Estoque

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-01  
**Bloqueia:** CARD-09, CARD-10

---

## Contexto

O serviço de Estoque é o segundo microserviço do sistema. Responsável por peças, quantidades e movimentações. Expõe um endpoint HTTP para consulta de disponibilidade (chamado sincronamente pelo Atendimento na abertura de OS) e consome eventos do RabbitMQ para realizar baixas de estoque quando uma OS é finalizada.

---

## Critérios de aceite

- [ ] Projeto criado com arquitetura hexagonal (Domain / Application / Infrastructure / API)
- [ ] Endpoint `GET /estoque/disponibilidade` funcional
- [ ] Consumer RabbitMQ `BaixaEstoqueConsumer` registrado mas aguardando CARD-09 para funcionar end-to-end
- [ ] `dotnet test` passa para os use cases de Estoque
- [ ] Serviço sobe via `docker-compose up` na porta 8081

---

## Projetos a criar

```
src/
└── Estoque/
    ├── OficinaMecanica.Estoque.Domain/
    ├── OficinaMecanica.Estoque.Application/
    ├── OficinaMecanica.Estoque.Infrastructure/
    └── OficinaMecanica.Estoque.API/
tests/
└── Estoque/
    ├── OficinaMecanica.Estoque.UnitTests/
    └── OficinaMecanica.Estoque.IntegrationTests/
```

---

## Domain

Entidades:
- `Peca` — Id, Nome, Descricao, Valor, QuantidadeEstoque
- `MovimentacaoEstoque` — Id, PecaId, Tipo (Entrada/Saida), Quantidade, Motivo, OcorridoEm

Enum:
- `TipoMovimentacao` — Entrada, Saida

---

## Application — Ports de saída

```
Application/Ports/Out/
├── IPecaRepository.cs
└── IMovimentacaoRepository.cs
```

## Application — Use Cases (TDD)

| Use Case | Descrição |
|---|---|
| `ConsultarDisponibilidadeUseCase` | Verifica se lista de peças tem quantidade suficiente |
| `BaixarEstoqueUseCase` | Subtrai quantidades e registra movimentação para cada peça |
| `GerenciarPecaUseCase` | CRUD de peças |

**Testes obrigatórios:**

`ConsultarDisponibilidadeUseCase`:
- Retorna `true` quando todas as peças têm quantidade suficiente
- Retorna `false` quando pelo menos uma peça está sem estoque

`BaixarEstoqueUseCase`:
- Subtrai corretamente e registra movimentação
- Lança exceção se quantidade solicitada > estoque disponível (proteção contra race condition)
- É idempotente: processar o mesmo `osId` duas vezes não gera dupla baixa

---

## Infrastructure

```
Infrastructure/Adapters/Out/Persistence/
├── AppDbContext.cs (schema: estoque)
├── Repositories/
│   ├── PecaRepository.cs
│   └── MovimentacaoRepository.cs
└── Migrations/
```

---

## API — Endpoints

| Método | Rota | Use Case |
|---|---|---|
| POST | `/estoque/disponibilidade` | `ConsultarDisponibilidadeUseCase` |
| GET | `/estoque/pecas` | `GerenciarPecaUseCase` |
| GET | `/estoque/pecas/{id}` | `GerenciarPecaUseCase` |
| POST | `/estoque/pecas` | `GerenciarPecaUseCase` |
| PUT | `/estoque/pecas/{id}` | `GerenciarPecaUseCase` |
| DELETE | `/estoque/pecas/{id}` | `GerenciarPecaUseCase` |

O consumer RabbitMQ (`BaixaEstoqueConsumer`) é um **adapter de entrada** registrado como `IHostedService`, não um controller.

---

## Passos

1. Criar projetos e adicionar à solution (monorepo)
2. Implementar Domain
3. Implementar portas de saída no Application
4. Implementar use cases com TDD
5. Implementar Infrastructure (EF Core, schema `estoque`)
6. Implementar API com os endpoints
7. Criar `BaixaEstoqueConsumer` como stub (loga o evento recebido, sem processar ainda)
8. Adicionar serviço ao `docker-compose.yml` na porta 8081
