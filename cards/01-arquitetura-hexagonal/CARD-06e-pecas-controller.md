# CARD-06e — API: PecasController (CRUD completo)

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-06d  
**Bloqueia:** CARD-07

---

## Contexto

O legacy expõe `api/pecas` como controller independente com CRUD completo e listagem com filtro por nome. No contexto da arquitetura hexagonal do Atendimento, as peças são um catálogo de referência local (o estoque real fica no microsserviço Estoque, CARD-08). É necessário criar `GerenciarPecaUseCase` e `PecasController` com as rotas do legacy.

---

## Endpoints obrigatórios (baseado no legacy)

| Método | Rota | Roles | Descrição |
|---|---|---|---|
| GET | `/pecas` | Autenticado | Lista peças ativas. Aceita `?nome=` para filtrar |
| GET | `/pecas/{id}` | Autenticado | Obtém peça por ID |
| POST | `/pecas` | Admin | Cria nova peça no catálogo local |
| PUT | `/pecas/{id}` | Admin | Atualiza peça |
| DELETE | `/pecas/{id}` | Admin | Desativa peça (soft delete) |

---

## Use case a criar

### `GerenciarPecaUseCase`

```
Application/UseCases/Peca/
├── GerenciarPecaUseCase.cs
├── GerenciarPecaRequest.cs
└── PecaResponse.cs
```

**Métodos:**
```csharp
Task<IEnumerable<PecaResponse>> ObterTodosAsync(CancellationToken ct = default)
Task<IEnumerable<PecaResponse>> BuscarPorNomeAsync(string nome, CancellationToken ct = default)
Task<PecaResponse?> ObterPorIdAsync(Guid id, CancellationToken ct = default)
Task<PecaResponse> CriarAsync(CriarPecaRequest request, CancellationToken ct = default)
Task<PecaResponse> AtualizarAsync(Guid id, AtualizarPecaRequest request, CancellationToken ct = default)
Task DesativarAsync(Guid id, CancellationToken ct = default)
```

**Requests:**
```csharp
record CriarPecaRequest(string Nome, string Descricao, decimal Preco)
record AtualizarPecaRequest(string Nome, string Descricao, decimal Preco)
```

**Response:**
```csharp
record PecaResponse(Guid Id, string Nome, string Descricao, decimal Preco, bool Ativo)
```

**Porta de saída — expandir `IPecaRepository`:**

O `IPecaRepository` atual tem apenas `ExisteAsync` e `ObterNomeAsync`. É necessário que passe a estender `IRepository<Peca>` para ter `ObterTodosAsync`, `AdicionarAsync`, etc. Ou criar uma segunda interface. Opção recomendada: fazer `IPecaRepository : IRepository<Peca>` e adicionar os métodos específicos.

---

## Controller

```
API/Adapters/In/Http/PecasController.cs
```

```csharp
[ApiController]
[Route("pecas")]
[Authorize]
public class PecasController : ControllerBase
{
    // GET /pecas
    // GET /pecas?nome=x
    // GET /pecas/{id}
    // POST /pecas  [Authorize(Roles = "Admin")]
    // PUT /pecas/{id}  [Authorize(Roles = "Admin")]
    // DELETE /pecas/{id}  [Authorize(Roles = "Admin")]
}
```

---

## Registro de DI

```csharp
builder.Services.AddScoped<GerenciarPecaUseCase>();
```

---

## Critérios de aceite

- [ ] `GET /pecas` retorna 200 com lista de peças ativas
- [ ] `GET /pecas?nome=filtro` retorna 200 filtrando por nome
- [ ] `GET /pecas/{id}` retorna 200 ou 404
- [ ] `POST /pecas` retorna 201 (Admin) ou 403
- [ ] `PUT /pecas/{id}` retorna 200 ou 404
- [ ] `DELETE /pecas/{id}` retorna 204 ou 404
- [ ] `dotnet build` sem erros ou warnings

---

## Passos

1. Expandir `IPecaRepository` para estender `IRepository<Peca>` e adicionar `BuscarPorNomeAsync`
2. Atualizar `PecaRepository` para implementar os novos métodos
3. Criar `GerenciarPecaUseCase` com todos os métodos
4. Criar `CriarPecaRequest`, `AtualizarPecaRequest`, `PecaResponse`
5. Criar `PecasController`
6. Registrar no `Program.cs`
7. Validar no Swagger
