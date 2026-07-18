# CARD-06d — API: ServicosController (CRUD completo)

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-06  
**Bloqueia:** CARD-07

---

## Contexto

O CARD-06 não criou um controller dedicado para serviços — usou um `CatalogoController` com rota `/catalogo/servicos` que não existe no legacy. O legacy expõe `api/servicos` como controller independente. É necessário remover o `CatalogoController`, criar `ServicosController` com as rotas corretas e expandir o `GerenciarCatalogoUseCase` com os métodos de leitura que faltam.

---

## Endpoints obrigatórios (baseado no legacy)

| Método | Rota | Roles | Descrição |
|---|---|---|---|
| GET | `/servicos` | Autenticado | Lista todos os serviços ativos |
| GET | `/servicos/{id}` | Autenticado | Obtém serviço por ID |
| POST | `/servicos` | Admin | Cria novo serviço |
| PUT | `/servicos/{id}` | Admin | Atualiza serviço |
| DELETE | `/servicos/{id}` | Admin | Desativa serviço (soft delete) |

---

## Ações na camada Application

### `GerenciarCatalogoUseCase` — adicionar métodos de leitura:

```csharp
Task<IEnumerable<ServicoResponse>> ObterServicosAsync(CancellationToken ct = default)
Task<ServicoResponse?> ObterServicoPorIdAsync(Guid id, CancellationToken ct = default)
```

**Response a criar:**
```csharp
record ServicoResponse(Guid Id, string Nome, string Descricao, decimal Preco, int TempoConclusaoMinutos, bool Ativo)
```

---

## Ações na camada API

1. **Remover** `CatalogoController.cs` (rota `/catalogo/*` não existe no legacy)
2. **Criar** `ServicosController.cs` com rota base `/servicos`

```csharp
[ApiController]
[Route("servicos")]
[Authorize]
public class ServicosController : ControllerBase
{
    // GET /servicos
    // GET /servicos/{id}
    // POST /servicos  [Authorize(Roles = "Admin")]
    // PUT /servicos/{id}  [Authorize(Roles = "Admin")]
    // DELETE /servicos/{id}  [Authorize(Roles = "Admin")]
}
```

---

## Critérios de aceite

- [ ] `CatalogoController` removido
- [ ] `GET /servicos` retorna 200 com lista de serviços ativos
- [ ] `GET /servicos/{id}` retorna 200 ou 404
- [ ] `POST /servicos` retorna 201 (Admin) ou 403
- [ ] `PUT /servicos/{id}` retorna 200 ou 404
- [ ] `DELETE /servicos/{id}` retorna 204 ou 404
- [ ] `dotnet build` sem erros ou warnings

---

## Passos

1. Remover `CatalogoController.cs`
2. Adicionar `ObterServicosAsync` e `ObterServicoPorIdAsync` ao `GerenciarCatalogoUseCase`
3. Criar `ServicoResponse`
4. Criar `ServicosController` com todos os endpoints
5. Validar no Swagger
