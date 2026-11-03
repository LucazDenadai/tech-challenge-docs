# CARD-44 — Qualidade: vulnerabilidades do SonarCloud e cobertura máxima

**Tipo:** Qualidade / Testes
**Status:** Concluído — PRs mergeados nos três serviços (2026-11-03)
**Depende de:** CARD-41a, CARD-43d
**Bloqueia:** —
**Repositórios alvo:** `tech-challenge-os`, `tech-challenge-billing`, `tech-challenge-operacoes`, `tech-challenge-docs`
**Decisão arquitetural:** [ADR-022](../../adr/ADR-022-observabilidade-fase4-stack-otel-local.md) (OTLP interno do cluster)

---

## Contexto

A primeira análise do SonarCloud (CARD-43d) mostrou três vulnerabilidades por serviço. Duas já foram corrigidas no CARD-43d (action fixada por SHA e `GITHUB_TOKEN` por variável). Sobra a S5332: `http://` no endpoint padrão do Jaeger.

A cobertura passa do gate de 80% (CARD-41a), mas não cobre tudo. O time quer chegar o mais perto possível de 100%, sem mudar funcionalidades.

## Ponto de partida (2026-11-03, último CI verde da `main`)

| Serviço | Linhas | Branches | Classes com falta |
|---|---|---|---|
| OS | 93,6% (1219/1301) | 86,3% (474/549) | 30 |
| Billing | 94,8% (1060/1118) | 82,4% (357/433) | 30 |
| Operações | 97,1% (966/994) | 84,4% (233/276) | 22 |

## Decisões (2026-11-03)

- **S5332 sem mudar comportamento:** o `appsettings.json` dos três serviços sempre define `Jaeger:Endpoint`, e o Kubernetes e o Compose sobrescrevem esse valor. O fallback `?? "http://jaeger:4318"` no código nunca é usado: ele sai. O tráfego OTLP continua em HTTP dentro do cluster, porque TLS no coletor seria mudança de infraestrutura (ADR-022).
- **Nenhuma mudança de funcionalidade:** testes novos não alteram regras, rotas, contratos nem mensagens. Uma alteração em código de produção só é aceita quando remove código inalcançável e os testes existentes continuam verdes sem mudança.
- **Sem teste falso:** cada teste novo verifica um resultado (retorno, estado, exceção, mensagem publicada). Teste que só executa a linha, sem asserção, não conta.
- **O que fica descoberto é registrado:** branch gerado pelo compilador ou caminho impossível de provocar sem alterar o código fica descoberto e listado aqui, com o motivo.
- **Gate:** continua em 80%. Os números novos viram a evidência no README; subir o gate fica para depois da avaliação, para não travar a entrega.
- **Testes locais no container do SDK:** o Smart App Control do host bloqueia as DLLs de teste (`mcr.microsoft.com/dotnet/sdk:10.0`, com Testcontainers pelo socket do Docker).

## Implementação (2026-11-03)

PRs de código: `tech-challenge-operacoes` #12, `tech-challenge-billing` #15 e `tech-challenge-os` #14. READMEs: #13, #16 e #15, na mesma ordem.

### A medição estava errada

`coverlet.runsettings` excluía `CompilerGeneratedAttribute`. O compilador gera uma classe com esse atributo para cada método `async`, então controllers, casos de uso, repositórios e consumidores ficavam fora da conta (no Operações, 112 de 113 métodos `async`). O atributo saiu da exclusão nos três serviços.

| Serviço | Antes (sem async) | Medição correta, antes dos testes | Depois |
|---|---|---|---|
| Operações | 97,1% / 84,4% | 95,6% / 88,7% | **99,6% / 98,4%** (CI) |
| Billing | 94,8% / 82,4% | 93,4% / 82,1% | **99,5% / 98,8%** (CI) |
| OS | 93,6% / 86,3% | 85,8% / **79,6%** | **97,2% / 89,6%** (CI) |

Linhas / branches. O OS medido corretamente ficava **abaixo do gate de 80%**: a correção da medição e os testes entraram no mesmo PR.

### Vulnerabilidades

- S7637 e S7636: corrigidas no CARD-43d (action por SHA, `GITHUB_TOKEN` por variável).
- S5332: o fallback `http://jaeger:4318` saiu do código; sem `Jaeger:Endpoint`, o serviço não sobe. O `appsettings.json` sempre define o endpoint, então nada muda em execução. O tráfego OTLP continua em HTTP dentro do cluster (ADR-022).

### Código de produção alterado

| Mudança | Por quê |
|---|---|
| `mensagem is null` removido no leitor (Operações, Billing) | A raiz já é um objeto JSON: a desserialização nunca devolve null |
| Consultas de OS por cliente e por status removidas (OS) | Sem uso em nenhum lugar |
| `ExtrairContextoTrace` `private` → `internal` (OS) e `InternalsVisibleTo` nos projetos | Testar guardas internas sem mudar a API pública |
| **Defeito:** documento de cliente desativado devolvia 500 (OS) | `DocumentoExisteAsync` só via clientes ativos; o ramo de reativação nunca rodava. Corrigido por decisão do time |
| **Defeito:** Mecânico recebia 403 ao ler veículo (OS) | `[Authorize]` da classe soma com o do método. A classe admite o Mecânico e as rotas que não são dele restringem a Admin e Atendente. Corrigido por decisão do time |

### Testes que mudaram de natureza

- **Concorrência determinística (Billing):** os testes paralelos de decisão e de notificação dependiam do agendamento; na execução medida, a nova tentativa não rodava. Um repositório decorado faz a gravação concorrente acontecer entre a leitura e o `SaveChanges`.
- **Eventos publicados validados (Billing):** o Billing não tinha o equivalente ao teste do Operações que passa cada evento pelo próprio validador.

### O que fica descoberto

| Onde | Motivo |
|---|---|
| `default` de `switch` sobre tipo aberto (resultado de reserva, decisão de orçamento, mensagem sem tratamento) | Inalcançável, e o compilador exige o caminho |
| Saída normal dos laços (despachantes, consumidor, expiração, reconciliação) | O laço só termina por cancelamento, que sai pela exceção do `Delay` |
| Falha ao publicar na DLQ e `ObjectDisposedException` ao fechar o canal | Exigem falha do broker no meio da operação |
| DynamoDB: tabela fora de `ACTIVE`, paginação da fila, `TransactionCanceled` sem motivo de condição | O DynamoDB Local não reproduz |
| Mensagem de erro do broker com mais de 500 caracteres | O broker não produz |
| `EmailSmtpAdapter`: autenticar e enviar | STARTTLS com certificado confiável e cliente SMTP não injetável |
| OS: parte dos branches da Saga, dos contratos e dos casos de uso | Não atacados por decisão do time (prazo da entrega) |

### Identificadores na DLQ (implementado a pedido do time)

Quando a falha transitória persistia, a mensagem ia para a DLQ com `x-message-id` e `x-correlation-id` vazios, embora o corpo levasse os dois. Agora o consumidor lê os identificadores do corpo com a leitura tolerante do `LeitorMensagemSaga` e preenche os cabeçalhos, o que facilita o runbook do alerta de DLQ. PRs: `tech-challenge-os` #16, `tech-challenge-billing` #17 e `tech-challenge-operacoes` #14.

## Critérios de aceite

- [x] Nenhuma vulnerabilidade aberta no SonarCloud nos três projetos.
- [x] Cobertura de linhas e branches o mais perto possível de 100% em cada serviço; o que ficar descoberto está listado com o motivo.
- [x] Nenhum teste existente alterado para passar; BDD ponta a ponta verde.
- [ ] READMEs e matriz com os números novos.

## Passos

1. Operações: S5332 e testes de cobertura.
2. Billing: S5332 e testes de cobertura.
3. OS: S5332 e testes de cobertura.
4. Registrar números, o que ficou descoberto e a análise do SonarCloud; atualizar READMEs e matriz.
