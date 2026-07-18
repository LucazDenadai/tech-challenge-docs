# CARD-06 — API: adapters de entrada (controllers)

**Tipo:** Refactor  
**Status:** To Do  
**Depende de:** CARD-04, CARD-05  
**Bloqueia:** CARD-07

---

## Contexto

A camada API é o adapter de entrada da arquitetura hexagonal. Os controllers não contêm lógica de negócio — apenas recebem a requisição HTTP, delegam ao use case via interface e retornam a resposta. O `Program.cs` registra todas as dependências.

---

## Critérios de aceite

- [ ] Controllers chamam use cases via interface (sem `new UseCase()` direto)
- [ ] Nenhuma lógica de negócio nos controllers
- [ ] Swagger documentado e acessível em `/swagger`
- [ ] JWT Bearer configurado
- [ ] Rate limiting no endpoint de login (10 req/min por IP)
- [ ] Global exception filter retornando respostas padronizadas
- [ ] `docker-compose up` sobe e Swagger responde
- [ ] Todos os endpoints da spec abaixo retornam os status HTTP corretos

---

## Endpoints obrigatórios (requisito do desafio)

| Método | Rota | Use Case | Descrição |
|---|---|---|---|
| POST | `/auth/login` | `AuthUseCase` | Login, retorna JWT |
| POST | `/ordens-servico` | `AbrirOrdemServicoUseCase` | Abertura de OS |
| GET | `/ordens-servico/{id}/status` | `ConsultarStatusOSUseCase` | Consulta status da OS |
| PUT | `/ordens-servico/{id}/orcamento` | `AprovarOrcamentoUseCase` | Aprovação/recusa de orçamento |
| GET | `/ordens-servico` | `ListarOrdensServicoUseCase` | Listagem ordenada |
| PUT | `/ordens-servico/{id}/status` | `AtualizarStatusOSUseCase` | Atualização de status |
| GET/POST/PUT/DELETE | `/clientes` | `GerenciarClienteUseCase` | CRUD clientes |
| GET/POST/PUT/DELETE | `/veiculos` | `GerenciarVeiculoUseCase` | CRUD veículos |
| GET/POST/PUT/DELETE | `/catalogo/servicos` | `GerenciarCatalogoUseCase` | CRUD serviços |
| GET/POST/PUT/DELETE | `/catalogo/pecas` | `GerenciarCatalogoUseCase` | CRUD peças |

---

## Estrutura de pastas

```
API/
├── Adapters/
│   └── In/
│       └── Http/
│           ├── OrdensServicoController.cs
│           ├── ClientesController.cs
│           ├── VeiculosController.cs
│           ├── CatalogoController.cs
│           └── AuthController.cs
├── Filters/
│   └── GlobalExceptionFilter.cs
└── Program.cs
```

---

## Registro de DI no Program.cs

```csharp
// Use cases
builder.Services.AddScoped<IAbrirOrdemServicoUseCase, AbrirOrdemServicoUseCase>();
builder.Services.AddScoped<IConsultarStatusOSUseCase, ConsultarStatusOSUseCase>();
// ... demais use cases

// Repositórios
builder.Services.AddScoped<IOrdemServicoRepository, OrdemServicoRepository>();
// ... demais repositórios

// Stubs (substituídos em cards futuros)
builder.Services.AddScoped<IEventPublisher, EventPublisherStub>();
builder.Services.AddScoped<IEstoquePort, EstoqueHttpStub>();
builder.Services.AddScoped<IEmailPort, EmailStub>();
```

---

## Passos

1. Criar controllers com injeção dos use cases via interface
2. Configurar `Program.cs` com todos os registros de DI
3. Configurar JWT, Swagger, Rate Limiting e CORS
4. Implementar `GlobalExceptionFilter`
5. Validar todos os endpoints no Swagger
