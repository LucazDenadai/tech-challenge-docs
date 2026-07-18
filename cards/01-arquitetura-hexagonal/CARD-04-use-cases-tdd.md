# CARD-04 — Application: use cases (TDD)

**Tipo:** Feature + TDD  
**Status:** To Do  
**Depende de:** CARD-03  
**Bloqueia:** CARD-06

---

## Contexto

Os use cases são o coração do Application. Cada um representa um fluxo de negócio único. Vamos implementá-los com TDD: escrever o teste primeiro, ver falhar, implementar, ver passar. As dependências externas (repositórios, publishers) são mockadas — nenhum banco necessário nesta etapa.

---

## Critérios de aceite

- [ ] Um use case por fluxo (sem classes "service" genéricas)
- [ ] Cada use case tem ao menos: teste do caminho feliz + teste do principal caminho de erro
- [ ] Todos os testes passam com `dotnet test`
- [ ] Use cases dependem apenas de interfaces (portas de saída do CARD-03)
- [ ] Nenhum `new` de repositório ou adapter dentro do use case (injeção de dependência)

---

## Use cases a implementar — na ordem sugerida

### 1. `ConsultarStatusOSUseCase`
Começa aqui por ser só leitura, sem side effects. Bom para calibrar o ritmo do TDD.

**Testes:**
- Retorna status e histórico quando OS existe
- Lança `NotFoundException` quando OS não encontrada

---

### 2. `AbrirOrdemServicoUseCase`
Fluxo principal do sistema. Mais complexo — valida disponibilidade de peças via `IEstoquePort`.

**Testes:**
- Cria OS com status `Recebida` e retorna o ID
- Lança exceção quando `IEstoquePort` indica indisponibilidade
- Lança exceção quando veículo não pertence ao cliente informado
- Persiste `HistoricoStatusOS` na criação

---

### 3. `AprovarOrcamentoUseCase`
Transição de estado: `AguardandoAprovacao` → `EmExecucao` (aprovado) ou `Cancelada` (recusado).

**Testes:**
- Aprovação muda status para `EmExecucao`
- Recusa muda status para `Cancelada`
- Lança exceção se OS não estiver em `AguardandoAprovacao`

---

### 4. `AtualizarStatusOSUseCase`
Avança o status da OS conforme a state machine. Quando status é `Finalizada`, publica evento via `IEventPublisher`.

**Testes:**
- Avança status na sequência correta
- Lança exceção para transição inválida
- Chama `IEventPublisher.PublishAsync` quando status muda para `Finalizada`
- Chama `IEmailPort.EnviarAtualizacaoStatusAsync` em qualquer mudança de status

---

### 5. `ListarOrdensServicoUseCase`
Ordenação específica: `EmExecucao` > `AguardandoAprovacao` > `EmDiagnostico` > `Recebida`. Mais antigas primeiro dentro do mesmo status. Exclui `Finalizada` e `Entregue`.

**Testes:**
- Retorna OS na ordem correta de status
- OS com mesmo status ordenadas por data de abertura (mais antiga primeiro)
- Não retorna OS com status `Finalizada` ou `Entregue`

---

### 6. `GerenciarClienteUseCase`
CRUD de clientes. Mecânico, sem surpresas.

**Testes:**
- Criar cliente com CPF válido
- Criar cliente com CNPJ válido
- Lança exceção para documento duplicado
- Soft delete marca `Ativo = false`

---

### 7. `GerenciarVeiculoUseCase`
CRUD de veículos.

**Testes:**
- Criar veículo com placa válida
- Lança exceção para placa duplicada
- Lança exceção quando cliente não existe

---

### 8. `GerenciarCatalogoUseCase`
CRUD de serviços e peças.

**Testes:**
- Criar serviço e peça com sucesso
- Atualizar preço de serviço
- Lança exceção ao tentar remover peça com estoque > 0

---

### 9. `AuthUseCase`
Por último — é infraestrutura de suporte.

**Testes:**
- Retorna JWT válido para credenciais corretas
- Lança exceção para senha incorreta
- Lança exceção para usuário não encontrado

---

## Estrutura de pastas

```
Application/
├── Ports/
│   └── Out/              // interfaces (CARD-03)
└── UseCases/
    ├── OrdemServico/
    │   ├── AbrirOrdemServicoUseCase.cs
    │   ├── ConsultarStatusOSUseCase.cs
    │   ├── AprovarOrcamentoUseCase.cs
    │   ├── AtualizarStatusOSUseCase.cs
    │   └── ListarOrdensServicoUseCase.cs
    ├── Cliente/
    │   └── GerenciarClienteUseCase.cs
    ├── Veiculo/
    │   └── GerenciarVeiculoUseCase.cs
    ├── Catalogo/
    │   └── GerenciarCatalogoUseCase.cs
    └── Auth/
        └── AuthUseCase.cs
```

## DTOs

Cada use case recebe um `Request` e retorna um `Response` — sem expor entidades do Domain para fora do Application.

```
Application/UseCases/OrdemServico/
├── AbrirOrdemServicoRequest.cs
├── AbrirOrdemServicoResponse.cs
├── ConsultarStatusOSResponse.cs
├── ListarOrdensServicoResponse.cs
└── ...
```

---

## Passos

1. Para cada use case na ordem acima:
   a. Escrever os testes no projeto `UnitTests` (todos vermelhos)
   b. Criar o use case com implementação mínima para passar
   c. Refatorar se necessário (sem quebrar os testes)
2. Rodar `dotnet test` ao final — 100% verde
