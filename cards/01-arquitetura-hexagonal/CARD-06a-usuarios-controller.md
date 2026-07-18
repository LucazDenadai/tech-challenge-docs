# CARD-06a — API: UsuariosController (CRUD completo)

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-06  
**Bloqueia:** CARD-07

---

## Contexto

O legacy expõe um CRUD completo de usuários (`api/usuarios`) restrito ao perfil Admin. Na refatoração hexagonal, o CARD-06 não incluiu este controller. É necessário criar o `GerenciarUsuarioUseCase` na camada Application e o `UsuariosController` na API.

---

## Endpoints obrigatórios (baseado no legacy)

| Método | Rota | Roles | Descrição |
|---|---|---|---|
| GET | `/usuarios` | Admin | Lista todos os usuários ativos. Aceita `?email=` para filtrar |
| GET | `/usuarios/{id}` | Admin | Obtém usuário por ID |
| POST | `/usuarios` | Admin | Cria novo usuário |
| PUT | `/usuarios/{id}` | Admin | Atualiza nome, email, perfil e senha |
| DELETE | `/usuarios/{id}` | Admin | Soft delete (Ativo = false) |

---

## Use case a criar

### `GerenciarUsuarioUseCase`

```
Application/UseCases/Usuario/
├── GerenciarUsuarioUseCase.cs
├── GerenciarUsuarioRequest.cs   (CriarUsuarioRequest, AtualizarUsuarioRequest)
└── UsuarioResponse.cs
```

**Métodos:**
```csharp
Task<IEnumerable<UsuarioResponse>> ObterTodosAsync(CancellationToken ct = default)
Task<UsuarioResponse?> ObterPorIdAsync(Guid id, CancellationToken ct = default)
Task<IEnumerable<UsuarioResponse>> BuscarPorEmailAsync(string email, CancellationToken ct = default)
Task<UsuarioResponse> CriarAsync(CriarUsuarioRequest request, CancellationToken ct = default)
Task<UsuarioResponse> AtualizarAsync(Guid id, AtualizarUsuarioRequest request, CancellationToken ct = default)
Task DesativarAsync(Guid id, CancellationToken ct = default)
```

**Regras de negócio:**
- Email único — lança `InvalidOperationException` se duplicado
- Senha armazenada como hash BCrypt
- `DesativarAsync` lança `NotFoundException` se usuário não existe

---

## Controller

```
API/Adapters/In/Http/UsuariosController.cs
```

- Rota base: `/usuarios`
- Todos os endpoints requerem `[Authorize(Roles = "Admin")]`
- Delega 100% ao `GerenciarUsuarioUseCase` — sem lógica no controller

---

## Registro de DI

```csharp
builder.Services.AddScoped<GerenciarUsuarioUseCase>();
```

---

## Critérios de aceite

- [ ] `GET /usuarios` retorna 200 com lista (vazia se nenhum)
- [ ] `GET /usuarios?email=x` retorna 200 filtrando por email
- [ ] `GET /usuarios/{id}` retorna 200 ou 404
- [ ] `POST /usuarios` retorna 201 com o usuário criado
- [ ] `POST /usuarios` com email duplicado retorna 422
- [ ] `PUT /usuarios/{id}` retorna 200 ou 404
- [ ] `DELETE /usuarios/{id}` retorna 204 (soft delete) ou 404
- [ ] Todos os endpoints retornam 401 sem token e 403 sem role Admin
- [ ] `dotnet build` sem erros ou warnings

---

## Passos

1. Criar `GerenciarUsuarioUseCase` com todos os métodos
2. Criar `CriarUsuarioRequest`, `AtualizarUsuarioRequest`, `UsuarioResponse`
3. Criar `UsuariosController` com injeção via construtor
4. Registrar no `Program.cs`
5. Validar no Swagger com token Admin
