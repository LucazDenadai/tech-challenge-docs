# CARD-41a — CI por serviço: testes, cobertura, qualidade e imagem

**Tipo:** Plataforma / CI
**Status:** Concluído — PRs mergeados nos três serviços (2026-10-13); SonarCloud pendente de projetos e segredos do dono da conta
**Depende de:** CARD-37, CARD-38, CARD-39, CARD-40c
**Bloqueia:** CARD-41b, CARD-43
**Repositórios alvo:** `tech-challenge-os`, `tech-challenge-billing`, `tech-challenge-operacoes`
**Decisão arquitetural:** ADR-014, [ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md) (pipeline próprio por serviço); convenções de CI da Fase 3 ([CARD-27](../05-fase3-aws/CARD-27-cicd-multi-repo.md))

---

## Contexto

O enunciado exige pipeline independente por microsserviço, cobertura mínima de 80% por serviço e análise de qualidade no CI. Os três repositórios da Fase 4 não têm workflow de build e teste, nem segredos, nem proteção de branch. O único workflow é o do BDD ponta a ponta no OS ([CARD-40c](CARD-40c-ambiente-integrado-e-bdd.md)).

## Ponto de partida (2026-10-13)

- Cobertura medida com unitários e integração juntos, excluindo migrations, `Program.cs`, código gerado e `[ExcludeFromCodeCoverage]` (mesma política da Fase 3):

  | Serviço | Linhas | Branches |
  |---|---|---|
  | OS | 91,5% | 75,1% (390/519) |
  | Billing | 94,7% | 81,5% (341/418) |
  | Operações | 93,2% | 73,8% (186/252) |

- Os testes de integração usam Testcontainers (PostgreSQL, RabbitMQ e, em Operações, DynamoDB Local); rodam no `ubuntu-latest` do GitHub Actions, que tem Docker.
- A Fase 3 usava SonarCloud com `SONAR_TOKEN` e `SONAR_ORGANIZATION`, cobertura OpenCover e imagens no GHCR.

## Decisões de implementação (2026-10-13)

- **Cobertura (decidido pelo time):** o gate exige **linhas e branches ≥80%** por serviço, medidos sobre unitários e integração juntos, com a exclusão da política acima. O relatório (ReportGenerator) vai como artefato e no resumo do job; abaixo do limite, o job falha com a métrica e o valor.
- **Branches de OS e Operações:** antes de ligar o gate, testes novos cobrem os caminhos de erro e de borda que faltam para chegar a 80%.
- **Qualidade (decidido pelo time):** SonarCloud, um projeto por serviço. O passo roda quando `SONAR_TOKEN` e `SONAR_ORGANIZATION` existem no repositório; sem eles, o job registra um aviso e segue, e o gate de cobertura continua valendo. Criar os projetos e os segredos é ação do dono da conta.
- **Workflow:** `.github/workflows/ci.yml` em cada repositório, disparado por push na `main` e por pull request. Jobs: `build-e-testes` (restore, build, unitários e integração com cobertura, gate, Sonar) → `imagem` (só na `main`: build e push no GHCR com tag `sha-<commit>`, sem `latest`).
- **Contrato:** um passo compara a cópia `contratos/asyncapi-saga-os.yaml` com a spec do `tech-challenge-docs` (ignorando as duas linhas de cabeçalho) e falha se divergirem (pendência do CARD-37b, 38a e 39a).
- **Independência:** cada workflow só lê o próprio repositório (e o `tech-challenge-docs`, para a spec). Alterar um serviço não dispara o pipeline dos outros.
- **Proteção de branch (autorizada pelo time):** `main` dos três repositórios exige PR e o check `build-e-testes` verde; no OS também o `bdd`.

## Implementação (2026-10-13)

- **Repositórios:** `tech-challenge-os` PR #7, `tech-challenge-billing` PR #6 e `tech-challenge-operacoes` PR #5. Cada um ganhou `coverlet.runsettings` (a política de cobertura) e `.github/workflows/ci.yml`.
- **Testes novos para o gate de branches:**
  - OS: guardas da Saga (evento no estado errado ou de outra execução, pagamento ou reserva) e validadores de senha forte, CPF, CNPJ e placa. O `SenhaForteAttribute` estava com 0% dos branches cobertos.
  - Operações: os eventos que Operações publica passam no próprio validador e são recusados quando malformados (Operações não lê o que publica, então esses validadores nunca rodavam); regras de borda de execução e catálogo. `ExigirValor`, copiado do OS e sem uso, foi removido.
- **Bug encontrado pelo CI:** no runner do GitHub, `ExecutionCompleted` foi processado antes de `ExecutionStarted` (filas diferentes) e a Saga parou em `InExecution`. A Saga passou a aceitar conclusão e falha também em `StartingExecution`, com teste de integração que publica na ordem invertida.
- **YAML:** um `echo` com `: ` em escalar simples quebrou o parse do workflow; os três arquivos passam no `actionlint`.
- **Proteção de branch:** `main` dos três exige PR, check `build-e-testes` verde (no OS também `bdd`), vale para administradores, sem force push. Aprovações obrigatórias: 0, porque o GitHub não deixa aprovar o próprio PR.
- **SonarCloud:** o passo existe e roda quando `SONAR_TOKEN` e `SONAR_ORGANIZATION` estiverem nos repositórios; até lá, o job registra aviso. Chave do projeto: variável `SONAR_PROJECT_KEY` ou `<organização>_<repositório>`.

### Pendências

- **SonarCloud (ação do dono da conta):** criar os três projetos no SonarCloud (importando os repositórios) e cadastrar `SONAR_TOKEN` e `SONAR_ORGANIZATION` em cada repositório.
- **Runner:** `ubuntu-latest` migra para Ubuntu 26 a partir de 2026-10-19 (aviso do GitHub).

## Critérios de aceite

- [x] Existem três workflows verificáveis, um por serviço, e alteração em um serviço não publica os outros.
- [x] Cada pipeline executa restore, build, testes, análise de qualidade (quando configurada) e gate de cobertura próprio.
- [x] O relatório mostra linhas e branches por serviço; abaixo de 80% em qualquer uma, o workflow falha.
- [x] Imagens tagueadas por commit, sem `latest`.
- [x] A cópia da spec é verificada contra o `tech-challenge-docs`.
- [x] Branch principal exige PR e checks verdes.
- [x] Segredos ficam no GitHub e não aparecem nos logs.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: CI independente por serviço

  Cenário: Bloquear merge abaixo da cobertura mínima
    Dado que a cobertura de branches do serviço está abaixo de 80 por cento
    Quando o workflow de CI termina
    Então o gate de cobertura falha com a métrica e o valor
    E o check requerido impede o merge

  Cenário: Publicar imagem só do serviço alterado
    Dado que um PR altera somente Billing e é integrado
    Quando o pipeline da main termina
    Então só a imagem do Billing ganha a tag do commit
```

## Passos

1. Testes para subir branches de OS e Operações acima de 80%.
2. Configuração de cobertura (`coverlet.runsettings`) e gate por serviço.
3. Workflow `ci.yml` nos três repositórios, com Sonar opcional e verificação da spec.
4. Imagem no GHCR com tag do commit.
5. Proteção de branch e prova do check obrigatório.

## Evidências

- URLs das execuções dos três workflows e do relatório de cobertura por serviço.
- Configuração de proteção de branch.

Registradas em 2026-10-13:

| Serviço | Linhas | Branches | Execução na `main` | Imagem |
|---|---|---|---|---|
| OS | 94% | 86,7% | [CI](https://github.com/LucazDenadai/tech-challenge-os/actions/runs/38463664642) | `ghcr.io/lucazdenadai/tech-challenge-os:sha-4fa79a3…` |
| Billing | 94,6% | 81,9% | [CI](https://github.com/LucazDenadai/tech-challenge-billing/actions/runs/38463680728) | `ghcr.io/lucazdenadai/tech-challenge-billing:sha-9cfd2f5…` |
| Operações | 97% | 83,8% | [CI](https://github.com/LucazDenadai/tech-challenge-operacoes/actions/runs/38463695178) | `ghcr.io/lucazdenadai/tech-challenge-operacoes:sha-4588717…` |

| Evidência | Resultado |
|---|---|
| Gate abaixo do mínimo | PR de teste no Billing com mínimo de 99%: [CI falhou](https://github.com/LucazDenadai/tech-challenge-billing/actions/runs/38464576256) com `Cobertura de linhas 94.6% abaixo de 99%` e o GitHub recusou o merge; PR fechado sem merge |
| Push direto na `main` | Recusado: `GH006: Protected branch update failed` |
| Independência | Cada workflow publica só a imagem do próprio repositório |
