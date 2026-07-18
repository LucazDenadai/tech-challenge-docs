# CARD-08f — Estoque: API — controllers + DI

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-08e  
**Bloqueia:** CARD-08g

---

## Contexto

Camada de entrada HTTP do microserviço de Estoque. Controllers são adapters puros — sem lógica de negócio, apenas delegam ao use case e mapeiam DTOs. A configuração de DI registra todos os use cases e repositórios.

---

## Critérios de aceite

- [ ] Todos os endpoints listados abaixo respondendo corretamente
- [ ] Controllers sem lógica de negócio
- [ ] DI configurada em `Program.cs`
- [ ] Swagger disponível em desenvolvimento
- [ ] `dotnet run` sobe na porta 8081 sem erros

---

## Endpoints

| Método | Rota | Use Case | Status de sucesso |
|---|---|---|---|
| POST | `/estoque/disponibilidade` | `ConsultarDisponibilidadeUseCase` | 200 |
| GET | `/estoque/pecas` | `GerenciarPecaUseCase` | 200 |
| GET | `/estoque/pecas/{id}` | `GerenciarPecaUseCase` | 200 / 404 |
| POST | `/estoque/pecas` | `GerenciarPecaUseCase` | 201 |
| PUT | `/estoque/pecas/{id}` | `GerenciarPecaUseCase` | 204 |
| DELETE | `/estoque/pecas/{id}` | `GerenciarPecaUseCase` | 204 |

---

## DTOs

### Request — `DisponibilidadeRequest`
```csharp
public record DisponibilidadeRequest(List<ItemDisponibilidade> Itens);
public record ItemDisponibilidade(Guid PecaId, int QuantidadeSolicitada);
```

### Response — `DisponibilidadeResponse`
```csharp
public record DisponibilidadeResponse(bool Disponivel);
```

---

## Estrutura de pastas

```
API/
├── Controllers/
│   ├── DisponibilidadeController.cs
│   └── PecasController.cs
├── DTOs/
│   ├── DisponibilidadeRequest.cs
│   └── PecaDto.cs
└── Program.cs
```

---

## Passos

1. Criar `PecasController` com os 5 endpoints CRUD
2. Criar `DisponibilidadeController` com o endpoint POST
3. Configurar DI em `Program.cs` (use cases, repositórios, DbContext)
4. Configurar porta 8081 em `launchSettings.json`
5. Adicionar Swagger
6. `dotnet run` e validar endpoints via Swagger UI
