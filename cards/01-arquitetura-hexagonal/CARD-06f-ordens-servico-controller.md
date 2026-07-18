# CARD-06f — API: OrdensServicoController (endpoints faltantes)

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-06  
**Bloqueia:** CARD-07

---

## Contexto

O CARD-06 implementou apenas os endpoints de fluxo principal da OS (abrir, consultar status, aprovar orçamento, atualizar status, listar). O legacy expõe endpoints adicionais: obter OS por ID, adicionar itens (serviço/peça), cancelar item, acompanhamento público por número, e métricas de tempo. É necessário criar os use cases correspondentes e expandir o `OrdensServicoController`.

---

## Endpoints faltantes (baseado no legacy)

| Método | Rota | Roles | Descrição |
|---|---|---|---|
| GET | `/ordens-servico/{id}` | Admin, Atendente, Mecanico | Obtém OS completa por ID |
| GET | `/ordens-servico/acompanhar/{numero}` | **Anônimo** | Acompanhamento público por número da OS |
| POST | `/ordens-servico/{id}/servicos` | Admin, Mecanico | Adiciona serviço à OS |
| POST | `/ordens-servico/{id}/pecas` | Admin, Mecanico | Adiciona peça à OS |
| DELETE | `/ordens-servico/{id}/itens/{itemId}` | Admin, Mecanico | Cancela item da OS (só em EmDiagnostico) |
| GET | `/ordens-servico/tempo-medio` | Admin, Atendente | Tempo médio global de execução (horas) |
| GET | `/ordens-servico/{numero}/tempo` | Admin, Atendente, Mecanico | Tempo de execução individual da OS |

---

## Use cases a criar

### `ObterOrdemServicoUseCase`

```csharp
Task<OrdemServicoDetalheResponse?> ExecutarAsync(Guid id, CancellationToken ct = default)
```

**Response:**
```csharp
record OrdemServicoDetalheResponse(
    Guid Id, string Numero, StatusOrdemServico Status,
    Guid ClienteId, Guid VeiculoId,
    string Observacoes, DateTime DataAbertura, DateTime? DataFechamento,
    decimal ValorTotal,
    IReadOnlyCollection<ItemServicoResponse> Servicos,
    IReadOnlyCollection<ItemPecaResponse> Pecas,
    IReadOnlyCollection<HistoricoStatusOSItem> Historico
)
record ItemServicoResponse(Guid Id, Guid ServicoId, int Quantidade, decimal ValorUnitario, decimal ValorTotal)
record ItemPecaResponse(Guid Id, Guid PecaId, int Quantidade, decimal ValorUnitario, decimal ValorTotal)
```

---

### `AcompanharOSUseCase`

Endpoint público — sem autenticação. Retorna dados da OS sem expor dados pessoais do cliente.

```csharp
Task<AcompanhamentoOSResponse?> ExecutarAsync(string numero, CancellationToken ct = default)
```

**Response:**
```csharp
record AcompanhamentoOSResponse(
    string Numero, StatusOrdemServico Status,
    DateTime DataAbertura, DateTime? DataFechamento,
    IReadOnlyCollection<ItemServicoResponse> Servicos,
    IReadOnlyCollection<ItemPecaResponse> Pecas,
    IReadOnlyCollection<HistoricoStatusOSItem> Historico
)
```

---

### `AdicionarItemOSUseCase`

```csharp
Task<OrdemServicoDetalheResponse> AdicionarServicoAsync(Guid osId, Guid servicoId, CancellationToken ct = default)
Task<OrdemServicoDetalheResponse> AdicionarPecaAsync(Guid osId, Guid pecaId, int quantidade, CancellationToken ct = default)
```

**Regras:** Só permitido nos status `EmDiagnostico` ou `EmExecucao`. Delega validação à entidade `OrdemServico`.

---

### `CancelarItemOSUseCase`

```csharp
Task<OrdemServicoDetalheResponse> ExecutarAsync(Guid osId, Guid itemId, CancellationToken ct = default)
```

**Regra:** Só permitido no status `EmDiagnostico`. Tenta remover de `ItensServico` primeiro, depois `ItensPeca`.

---

### `ObterTempoExecucaoUseCase`

```csharp
Task<TempoMedioResponse> ObterTempoMedioAsync(CancellationToken ct = default)
Task<TempoIndividualResponse?> ObterTempoIndividualAsync(string numero, CancellationToken ct = default)
```

**Responses:**
```csharp
record TempoMedioResponse(double TempoMedioHoras, int TotalOSFinalizadas)
record TempoIndividualResponse(string Numero, StatusOrdemServico Status, double TempoDecorridoHoras, bool Finalizada)
```

**Referência de tempo:** 1 a 3 dias úteis (entre 8h e 24h). Para OS em andamento, calcula o tempo desde `DataAbertura` até agora.

---

## Expansão do controller existente

Adicionar ao `OrdensServicoController`:

```csharp
[HttpGet("{id:guid}")]                          // GET /ordens-servico/{id}
[HttpGet("acompanhar/{numero}")]                // GET /ordens-servico/acompanhar/{numero}  [AllowAnonymous]
[HttpPost("{id:guid}/servicos")]                // POST /ordens-servico/{id}/servicos
[HttpPost("{id:guid}/pecas")]                   // POST /ordens-servico/{id}/pecas
[HttpDelete("{id:guid}/itens/{itemId:guid}")]   // DELETE /ordens-servico/{id}/itens/{itemId}
[HttpGet("tempo-medio")]                        // GET /ordens-servico/tempo-medio
[HttpGet("{numero}/tempo")]                     // GET /ordens-servico/{numero}/tempo
```

---

## Registro de DI

```csharp
builder.Services.AddScoped<ObterOrdemServicoUseCase>();
builder.Services.AddScoped<AcompanharOSUseCase>();
builder.Services.AddScoped<AdicionarItemOSUseCase>();
builder.Services.AddScoped<CancelarItemOSUseCase>();
builder.Services.AddScoped<ObterTempoExecucaoUseCase>();
```

---

## Critérios de aceite

- [ ] `GET /ordens-servico/{id}` retorna 200 com OS completa (itens + histórico) ou 404
- [ ] `GET /ordens-servico/acompanhar/{numero}` retorna 200 sem autenticação ou 404
- [ ] `POST /ordens-servico/{id}/servicos` retorna 200 com OS atualizada ou 422
- [ ] `POST /ordens-servico/{id}/pecas` retorna 200 com OS atualizada ou 422
- [ ] `DELETE /ordens-servico/{id}/itens/{itemId}` retorna 200 com OS atualizada ou 422
- [ ] `GET /ordens-servico/tempo-medio` retorna 200 com média em horas
- [ ] `GET /ordens-servico/{numero}/tempo` retorna 200 ou 404
- [ ] `dotnet build` sem erros ou warnings

---

## Passos

1. Criar `ObterOrdemServicoUseCase` e response correspondente
2. Criar `AcompanharOSUseCase` (sem dados pessoais do cliente)
3. Criar `AdicionarItemOSUseCase` (serviço e peça)
4. Criar `CancelarItemOSUseCase`
5. Criar `ObterTempoExecucaoUseCase`
6. Expandir `OrdensServicoController` com os 7 endpoints faltantes
7. Registrar todos no `Program.cs`
8. Validar no Swagger
