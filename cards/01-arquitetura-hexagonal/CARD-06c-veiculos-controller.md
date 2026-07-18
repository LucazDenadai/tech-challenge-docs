# CARD-06c — API: VeiculosController (CRUD completo)

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-06  
**Bloqueia:** CARD-07

---

## Contexto

O CARD-06 criou o `VeiculosController` apenas com `POST`, `GET/{id}` (stub) e `PUT`. O legacy expõe `GET /veiculos` (listagem), `GET /veiculos/cliente/{id}`, `GET /veiculos/cliente/documento/{doc}` e `DELETE /veiculos/{id}`. É necessário expandir o use case e o controller existentes.

---

## Endpoints obrigatórios (baseado no legacy)

| Método | Rota | Roles | Descrição |
|---|---|---|---|
| GET | `/veiculos` | Admin, Atendente | Lista todos os veículos |
| GET | `/veiculos/{id}` | Admin, Atendente, Mecanico | Obtém veículo por ID |
| GET | `/veiculos/cliente/{clienteId}` | Admin, Atendente, Mecanico | Lista veículos de um cliente por ID |
| GET | `/veiculos/cliente/documento/{documento}` | Admin, Atendente | Lista veículos de um cliente por CPF/CNPJ |
| POST | `/veiculos` | Admin, Atendente | Cria veículo vinculado a um cliente |
| PUT | `/veiculos/{id}` | Admin, Atendente | Atualiza marca, modelo, ano, cor |
| DELETE | `/veiculos/{id}` | Admin | Remove veículo (erro se existem OS vinculadas) |

---

## Expansão do use case existente

### `GerenciarVeiculoUseCase` — adicionar métodos:

```csharp
Task<IEnumerable<VeiculoResponse>> ObterTodosAsync(CancellationToken ct = default)
Task<VeiculoResponse?> ObterPorIdAsync(Guid id, CancellationToken ct = default)
Task<IEnumerable<VeiculoResponse>> ObterPorClienteAsync(Guid clienteId, CancellationToken ct = default)
Task<IEnumerable<VeiculoResponse>> ObterPorDocumentoClienteAsync(string documento, CancellationToken ct = default)
Task RemoverAsync(Guid id, CancellationToken ct = default)
```

**Response a criar:**
```csharp
record VeiculoResponse(Guid Id, Guid ClienteId, string Placa, string Marca, string Modelo, int Ano, string Cor)
```

**Regra de negócio — RemoverAsync:**
- Lança `InvalidOperationException` se existir qualquer OS vinculada ao veículo (qualquer status)
- Hard delete (veículo não tem campo `Ativo`)

---

## Expansão do controller existente

Adicionar ao `VeiculosController`:

```csharp
[HttpGet]                                        // GET /veiculos
[HttpGet("{id:guid}")]                           // GET /veiculos/{id}  (já existe como stub — implementar)
[HttpGet("cliente/{clienteId:guid}")]            // GET /veiculos/cliente/{clienteId}
[HttpGet("cliente/documento/{documento}")]        // GET /veiculos/cliente/documento/{doc}
[HttpDelete("{id:guid}")]                        // DELETE /veiculos/{id}
```

---

## Critérios de aceite

- [ ] `GET /veiculos` retorna 200 com lista
- [ ] `GET /veiculos/{id}` retorna 200 ou 404
- [ ] `GET /veiculos/cliente/{clienteId}` retorna 200 com veículos do cliente
- [ ] `GET /veiculos/cliente/documento/{doc}` retorna 200 com veículos do cliente pelo CPF/CNPJ
- [ ] `POST /veiculos` retorna 201 com `Location: /veiculos/{id}`
- [ ] `POST /veiculos` com placa duplicada retorna 422
- [ ] `PUT /veiculos/{id}` retorna 200 ou 404
- [ ] `DELETE /veiculos/{id}` retorna 204 ou 400 (se tem OS) ou 404
- [ ] `dotnet build` sem erros ou warnings

---

## Passos

1. Adicionar os novos métodos ao `GerenciarVeiculoUseCase`
2. Criar `VeiculoResponse`
3. Expandir `VeiculosController` com os endpoints faltantes
4. Implementar `RemoverAsync` verificando OS vinculadas via `IOrdemServicoRepository`
5. Validar no Swagger
