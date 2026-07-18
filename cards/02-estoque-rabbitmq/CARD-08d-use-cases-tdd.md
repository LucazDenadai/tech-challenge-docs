# CARD-08d — Estoque: Application — use cases (TDD)

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-08c  
**Bloqueia:** CARD-08e

---

## Contexto

Implementação dos use cases com TDD: escrever o teste, ver falhar, implementar, ver passar. Os repositórios são mockados com Moq — sem banco nesta etapa.

---

## Critérios de aceite

- [ ] Todos os testes listados abaixo escritos **antes** da implementação
- [ ] Todos os testes passando após a implementação
- [ ] Use cases sem referência a EF Core ou Infrastructure
- [ ] `dotnet test` no projeto UnitTests passa sem erros

---

## Use Cases e testes obrigatórios

### `ConsultarDisponibilidadeUseCase`

Recebe lista de `{ PecaId, QuantidadeSolicitada }` e retorna `bool`.

| Teste | Cenário |
|---|---|
| `Retorna_True_QuandoTodasPecasTemEstoqueSuficiente` | Todas as peças têm quantidade ≥ solicitada |
| `Retorna_False_QuandoPeloMenosUmaPecaInsuficiente` | Uma peça com estoque menor que o solicitado |

### `BaixarEstoqueUseCase`

Recebe `osId` + lista de `{ PecaId, Quantidade }`. Subtrai e registra movimentação.

| Teste | Cenário |
|---|---|
| `SubtraiCorretamente_ERegistraMovimentacao` | Caminho feliz — subtrai e persiste movimentação com `OsId` |
| `Lanca_Excecao_QuandoQuantidadeSolicitadaMaiorQueEstoque` | Quantidade > estoque disponível |
| `Idempotente_QuandoOsIdJaProcessado` | Mesmo `osId` enviado duas vezes — segunda chamada não altera estoque nem cria movimentação |

### `GerenciarPecaUseCase`

CRUD de peças.

| Teste | Cenário |
|---|---|
| `CriarPeca_RetornaPecaCriada` | Peca criada com dados válidos |
| `ObterPecaPorId_Retorna_QuandoExiste` | Peca encontrada pelo Id |
| `ObterPecaPorId_Retorna_Null_QuandoNaoExiste` | Id inexistente retorna null |
| `AtualizarPeca_PersisteMudancas` | Dados atualizados corretamente |
| `RemoverPeca_ChamaRepositorio` | Repositório chamado com o Id correto |

---

## Estrutura de pastas

```
Application/
└── UseCases/
    ├── ConsultarDisponibilidadeUseCase.cs
    ├── BaixarEstoqueUseCase.cs
    └── GerenciarPecaUseCase.cs

tests/Estoque/OficinaMecanica.Estoque.UnitTests/
├── ConsultarDisponibilidadeUseCaseTests.cs
├── BaixarEstoqueUseCaseTests.cs
└── GerenciarPecaUseCaseTests.cs
```

---

## Passos

1. Instalar Moq no projeto UnitTests
2. Escrever todos os testes (vermelhos)
3. Implementar `ConsultarDisponibilidadeUseCase` até testes ficarem verdes
4. Implementar `BaixarEstoqueUseCase` com checagem de idempotência via `IMovimentacaoRepository.ExisteMovimentacaoPorOsIdAsync`
5. Implementar `GerenciarPecaUseCase`
6. `dotnet test` — todos verdes
