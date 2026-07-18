# ADR-004 — Estoque como fonte da verdade para peças

- **Status:** Aceito
- **Data:** 2026-05-27
- **Contexto:** Tech Challenge — Oficina Mecânica (microsserviços)

---

## Contexto

O serviço **Atendimento** mantinha um catálogo próprio de peças (`Peca`, `PecaRepository`, `PecasController`) utilizado para validar disponibilidade ao abrir uma OS e para compor os `ItensPeca` da ordem de serviço.

O serviço **Estoque** mantém seu próprio registro de peças com quantidade física, histórico de movimentações e o consumer que processa o evento `OsFinalizadaEvent` para dar baixa no estoque.

Ambos os cadastros geravam `Guid` independentes. Isso criava uma dependência operacional frágil: o operador precisava cadastrar a peça nos dois serviços e garantir manualmente que os IDs coincidissem para que a baixa automática funcionasse ao finalizar uma OS.

---

## Problema

IDs de peça gerados independentemente em dois serviços distintos tornam o vínculo entre `ItensPeca` (Atendimento) e `Peca` (Estoque) dependente de sincronização manual. Qualquer divergência resulta em falha silenciosa na baixa de estoque — o consumer não encontra a peça e a movimentação não é registrada.

---

## Decisão

O **Estoque é a única fonte da verdade para peças**.

- Peças são cadastradas exclusivamente via `POST /estoque/pecas`
- O Atendimento **não mantém catálogo próprio** de peças
- Ao abrir uma OS ou adicionar uma peça à OS, o Atendimento consulta o Estoque via `IEstoquePort` (porta de saída já existente) para obter nome, valor e validar disponibilidade
- O `PecaId` armazenado em `ItemPecaOS` (Atendimento) é sempre o ID do Estoque, garantindo o vínculo no consumer do `OsFinalizadaEvent`

### O que é removido do Atendimento

| Artefato | Motivo |
|---|---|
| `Peca` (domain entity) | Catálogo duplicado |
| `PecaConfiguration` | Configuração EF da entidade removida |
| `PecaRepository` / `IPecaRepository` | Repositório da entidade removida |
| `GerenciarPecaUseCase` (Atendimento) | Use case do catálogo removido |
| `PecasController` (Atendimento) | Endpoint do catálogo removido |

### O que é ajustado

| Artefato | Ajuste |
|---|---|
| `IEstoquePort` | Adicionar `ObterPecaAsync(Guid id)` para buscar nome e valor |
| `EstoqueHttpAdapter` | Implementar `ObterPecaAsync` |
| `AbrirOrdemServicoUseCase` | Usar `IEstoquePort` para validar disponibilidade e obter dados da peça |
| `AdicionarItemPecaUseCase` | Usar `IEstoquePort` para validar e obter dados da peça |

---

## Alternativas rejeitadas

### Manter dois cadastros com IDs sincronizados manualmente
Operacionalmente frágil. Um cadastro fora de sincronia quebra o fluxo de finalização sem qualquer alerta em tempo de compilação ou execução imediata.

### Atendimento como fonte da verdade (Estoque consulta Atendimento)
Inverte a direção do acoplamento de forma inadequada: Estoque é o domínio de controle físico de peças, não Atendimento. A baixa de estoque é responsabilidade do Estoque; faz sentido que ele seja a referência canônica.

### Banco de dados compartilhado
Viola o isolamento de microsserviços. Cada serviço deve ser dono do seu schema.

---

## Consequências

**Positivas:**
- Um único ponto de cadastro elimina inconsistências de ID
- `OsFinalizadaEvent` sempre carrega IDs válidos para o consumer do Estoque
- Reduz superfície de código no Atendimento (menos entidades, repositórios, controllers)

**Negativas/Riscos:**
- Atendimento passa a depender do Estoque em tempo de execução para abrir OS (chamada HTTP via `IEstoquePort`)
- Se o Estoque estiver indisponível, a abertura de OS falha — mitigado pelo circuit breaker Polly já configurado
- O stub `EstoqueHttpStub` precisa ser atualizado para simular `ObterPecaAsync`
