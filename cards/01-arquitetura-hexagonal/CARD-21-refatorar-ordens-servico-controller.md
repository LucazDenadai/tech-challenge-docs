# CARD-21 — Refatorar OrdensServicoController em controllers menores

**Tipo:** Refatoração  
**Status:** To Do  
**Depende de:** CARD-06f  
**Origem:** Sonar — Code Smell Major (brain-overload, asp.net)

---

## Contexto

O `OrdensServicoController` acumula 10 use cases injetados no construtor e expõe endpoints de domínios distintos (ciclo de vida da OS, itens, acompanhamento público, métricas de tempo). O Sonar aponta duas violações derivadas desse design:

- **Construtor com 10 parâmetros** (limite: 7) — `S107`
- **Controller com múltiplas responsabilidades** — sugere divisão em 9 controllers menores

A divisão resolve os dois issues simultaneamente e melhora a coesão das rotas.

---

## Divisão proposta

| Novo controller | Rota base | Use cases | Endpoints |
|---|---|---|---|
| `OrdensServicoController` | `/ordens-servico` | `AbrirOrdemServicoUseCase`, `ListarOrdensServicoUseCase`, `ObterOrdemServicoUseCase` | POST, GET (lista), GET `{id}` |
| `StatusOSController` | `/ordens-servico/{id}/status` | `ConsultarStatusOSUseCase`, `AtualizarStatusOSUseCase` | GET, PUT |
| `OrcamentoOSController` | `/ordens-servico/{id}/orcamento` | `AprovarOrcamentoUseCase` | PUT |
| `ItensOSController` | `/ordens-servico/{id}` | `AdicionarItemOSUseCase`, `CancelarItemOSUseCase` | POST `/servicos`, POST `/pecas`, DELETE `/itens/{itemId}` |
| `AcompanhamentoOSController` | `/ordens-servico/acompanhar` | `AcompanharOSUseCase` | GET `{numero}` (AllowAnonymous) |
| `TempoExecucaoOSController` | `/ordens-servico` | `ObterTempoExecucaoUseCase` | GET `tempo-medio`, GET `{numero}/tempo` |

---

## Critérios de aceite

- [ ] Nenhum controller tem mais de 7 parâmetros no construtor
- [ ] Todos os endpoints existentes continuam funcionando nas mesmas rotas
- [ ] `dotnet build` com 0 erros
- [ ] `dotnet test` 100% passando
- [ ] Sonar não aponta mais `S107` nem brain-overload no controller de OS

---

## Passos

1. Criar os novos controllers com as rotas e use cases mapeados acima
2. Remover os métodos correspondentes do `OrdensServicoController` original
3. Apagar o controller original quando estiver vazio
4. Ajustar registros de DI se necessário
5. Validar no Swagger que todas as rotas continuam presentes
