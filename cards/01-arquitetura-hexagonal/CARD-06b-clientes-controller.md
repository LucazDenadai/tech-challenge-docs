# CARD-06b — API: ClientesController (CRUD completo)

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-06  
**Bloqueia:** CARD-07

---

## Contexto

O CARD-06 criou o `ClientesController` apenas com `POST`, `PUT` e `DELETE`. O legacy expõe também `GET /clientes` (listagem com busca) e `GET /clientes/{id}`. É necessário expandir o use case e o controller existentes.

---

## Endpoints obrigatórios (baseado no legacy)

| Método | Rota | Roles | Descrição |
|---|---|---|---|
| GET | `/clientes` | Admin, Atendente | Lista clientes ativos. Aceita `?busca=` (nome, CPF/CNPJ ou e-mail) |
| GET | `/clientes/{id}` | Admin, Atendente | Obtém cliente por ID |
| POST | `/clientes` | Admin, Atendente | Cria cliente (reativa se CPF/CNPJ já existir inativo) |
| PUT | `/clientes/{id}` | Admin, Atendente | Atualiza nome, e-mail, telefone, endereço |
| DELETE | `/clientes/{id}` | Admin | Soft delete |

---

## Expansão do use case existente

### `GerenciarClienteUseCase` — adicionar métodos:

```csharp
Task<IEnumerable<ClienteResponse>> ObterTodosAsync(CancellationToken ct = default)
Task<ClienteResponse?> ObterPorIdAsync(Guid id, CancellationToken ct = default)
Task<IEnumerable<ClienteResponse>> BuscarAsync(string termo, CancellationToken ct = default)
```

**Response a criar:**
```csharp
record ClienteResponse(Guid Id, string Nome, string Documento, string Email, string Telefone, string Endereco, bool Ativo)
```

---

## Expansão do controller existente

Adicionar ao `ClientesController`:

```csharp
[HttpGet]           // GET /clientes?busca=
[HttpGet("{id}")]   // GET /clientes/{id}
```

---

## Critérios de aceite

- [ ] `GET /clientes` retorna 200 com lista de clientes ativos
- [ ] `GET /clientes?busca=joao` filtra por nome, CPF/CNPJ ou e-mail
- [ ] `GET /clientes/{id}` retorna 200 com dados ou 404
- [ ] `POST /clientes` retorna 201 com `Location: /clientes/{id}`
- [ ] `POST /clientes` com CPF inválido retorna 400
- [ ] `POST /clientes` com CPF duplicado ativo retorna 422
- [ ] `PUT /clientes/{id}` retorna 200 com dados atualizados ou 404
- [ ] `DELETE /clientes/{id}` retorna 204 (soft delete) ou 404
- [ ] `dotnet build` sem erros ou warnings

---

## Passos

1. Adicionar `ObterTodosAsync`, `ObterPorIdAsync`, `BuscarAsync` ao `GerenciarClienteUseCase`
2. Criar `ClienteResponse`
3. Adicionar `GET /clientes` e `GET /clientes/{id}` ao `ClientesController`
4. Validar no Swagger
