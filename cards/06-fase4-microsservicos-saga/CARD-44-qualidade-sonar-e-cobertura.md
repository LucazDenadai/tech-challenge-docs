# CARD-44 — Qualidade: vulnerabilidades do SonarCloud e cobertura máxima

**Tipo:** Qualidade / Testes
**Status:** Em andamento — escopo definido (2026-11-03)
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

## Critérios de aceite

- [ ] Nenhuma vulnerabilidade aberta no SonarCloud nos três projetos.
- [ ] Cobertura de linhas e branches o mais perto possível de 100% em cada serviço; o que ficar descoberto está listado com o motivo.
- [ ] Nenhum teste existente alterado para passar; BDD ponta a ponta verde.
- [ ] READMEs e matriz com os números novos.

## Passos

1. Operações: S5332 e testes de cobertura.
2. Billing: S5332 e testes de cobertura.
3. OS: S5332 e testes de cobertura.
4. Registrar números, o que ficou descoberto e a análise do SonarCloud; atualizar READMEs e matriz.
