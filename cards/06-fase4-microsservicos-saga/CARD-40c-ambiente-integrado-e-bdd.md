# CARD-40c — Ambiente integrado e BDD ponta a ponta

**Tipo:** Testes / Integração distribuída
**Status:** Concluído — PR #6 mergeado em `tech-challenge-os` (2026-10-13); BDD verde no CI; trace como evidência no CARD-42
**Depende de:** CARD-40a, CARD-40b
**Bloqueia:** CARD-42, CARD-43
**Repositório alvo:** `tech-challenge-os`, pasta `e2e/` (decidido pelo time em 2026-10-13)
**Decisão arquitetural:** [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md); [ADR-019](../../adr/ADR-019-integracao-mercado-pago-fase4.md) (Billing em modo `Simulado`)

---

## Contexto

O enunciado exige pelo menos um fluxo completo em BDD. Hoje cada serviço tem o próprio Compose com o próprio RabbitMQ, e não existe ferramenta de BDD nos repositórios. Este card sobe os três serviços com um broker só e automatiza os cenários Gherkin do [CARD-40](CARD-40-saga-e-bdd-integrado.md#cenários-bdd-obrigatórios).

**Por que no repositório do OS:** o OS é o orquestrador e dono da Saga, então o teste ponta a ponta mora com ele. Um repositório só para e2e separaria melhor as responsabilidades, mas criaria um sexto repositório com CI e índice próprios. O `tech-challenge-docs` não tem código nem CI de .NET.

## Ponto de partida (2026-10-13)

- Os três serviços têm testes de integração próprios, mas cada um simula os outros publicando eventos no broker. Nenhum teste sobe os três juntos.
- Nenhum repositório tem workflow de CI. Só o Billing tem `Dockerfile` (criado para o sandbox no CARD-39b).
- Os três validam o mesmo JWT (chave, issuer `oficina-atendimento`, audience `oficina-atendimento-api`). A filial de demonstração tem o mesmo Id nos seeds do OS e de Operações, e Operações tem saldo das peças do seed nela.

## Decisões de implementação (2026-10-13)

- **Ferramenta:** [Reqnroll](https://reqnroll.net/) com xUnit, decidido pelo time. Os cenários ficam em arquivos `.feature` (Gherkin) executáveis; os steps em C# chamam as APIs reais. Atende o requisito R-10 da [matriz](matriz-requisitos.md); não substitui os testes de integração de cada serviço.
- **Projeto:** `e2e/OficinaMecanica.E2E` em `tech-challenge-os`, fora da solution do serviço: o `dotnet test` do OS não precisa do ambiente integrado.
- **Ambiente:** `e2e/docker-compose.yml` com OS, Billing (`MercadoPago:Modo=Simulado`) e Operações, um RabbitMQ para os três, um PostgreSQL por serviço (ADR-016), DynamoDB Local e Jaeger. Os serviços são construídos a partir dos três repositórios lado a lado (`../../tech-challenge-*`), decidido pelo time; com imagens publicadas (CARD-41), basta trocar `build` por `image`. OS e Operações ganham `Dockerfile` no mesmo padrão do Billing.
- **Atores:** atendente, mecânico e Admin entram pelo login do OS (usuários do seed). O cliente usa um JWT no formato da Lambda de CPF (RFC-003: `role=Cliente`, `cliente_id`), assinado pelo teste com a chave do ambiente; a Lambda fica fora do e2e. O pagamento é confirmado pela rota do simulador do Billing (Admin), como faria o webhook.
- **Cenários em BDD:** o obrigatório ("Concluir fluxo de OS com pagamento aprovado") e os de rejeição de orçamento, falta de estoque, pagamento recusado e falha operacional depois do pagamento. Os cenários de pagamento desconhecido, reinício e webhook duplicado continuam como testes de integração nos serviços (CARD-39b, CARD-40a e CARD-40b), como o CARD-40 permite.
- **Execução:** `e2e/executar.sh` sobe o Compose, roda os testes em um container do SDK na rede do Compose (o Smart App Control da máquina de desenvolvimento bloqueia DLLs de teste no host) e derruba tudo. No CI, um workflow do OS faz checkout dos três repositórios públicos e chama o mesmo script; o relatório TRX vira artefato.
- **Trace:** o Jaeger do Compose recebe os traces dos três serviços; a captura do trace correlacionado como evidência fica no CARD-42.

## Escopo

- Compose em `e2e/` com OS, Billing (`MercadoPago:Modo=Simulado`), Operações, um RabbitMQ, os bancos de cada serviço e DynamoDB Local.
- Projeto BDD em Reqnroll com o cenário obrigatório e os cenários escolhidos do CARD-40.
- O pagamento é confirmado pela rota do simulador de Billing, como faria o webhook.
- Execução no CI do OS com dependências isoladas, com checkout dos três repositórios.

## Implementação (2026-10-13)

- **Repositórios:** `tech-challenge-os` PR #6 (Dockerfile, Compose integrado, projeto Reqnroll, workflow e README, em 6 commits), `tech-challenge-operacoes` PR #4 (Dockerfile) e `tech-challenge-billing` PR #5.
- **Billing expõe `referenciaProvedor`:** a rota do simulador recebe a referência do provedor, que a consulta de pagamento não devolvia; os testes do Billing a liam do banco. A consulta (`/billing/pagamentos/meus` e `/os/{osId}`) passou a trazer o campo, que é o ID da cobrança no Mercado Pago e não é dado do pagador. Assim o BDD usa só APIs públicas.
- **Estoque independente de ordem:** as verificações de estoque comparam com o saldo lido no início de cada cenário. Os cenários passaram duas vezes seguidas sobre o mesmo ambiente.
- **Esperas por condição:** os efeitos chegam por mensagem; cada step espera a condição (até 60 s) em vez de dormir um tempo fixo.
- **Script executável no CI:** o bit de execução do `e2e/executar.sh` não entra no Git pelo Windows; foi marcado no índice (`git update-index --chmod=+x`).
- **CI:** [execução do workflow](https://github.com/LucazDenadai/tech-challenge-os/actions/runs/38448697151) com os três repositórios: 5 de 5 cenários aprovados; relatório `bdd.trx` como artefato.

### Pendências

- **Trace correlacionado:** o Jaeger sobe no ambiente; a captura do trace como evidência fica no CARD-42.
- **Imagens publicadas:** quando o CARD-41 publicar imagens, o Compose troca `build` por `image`.
- **Runner:** o GitHub avisa que `ubuntu-latest` migra para Ubuntu 26 a partir de 2026-10-19; acompanhar no CARD-41.

## Critérios de aceite

- [x] Fluxo completo executável: abrir OS → diagnosticar → gerar/aprovar orçamento → reservar estoque por filial → cobrar/confirmar → executar → concluir → status e histórico da OS.
- [x] Cenários Gherkin automatizados rodam em CI com broker e bancos isolados.
- [ ] Logs/traces identificam a Saga completa; dados pessoais e segredos não aparecem na evidência. *(Jaeger no ambiente; captura no CARD-42.)*
- [x] O BDD não depende de ambiente cloud pago nem do sandbox do Mercado Pago.

## Cenários de aceite (Gherkin)

Os cenários são os do [CARD-40](CARD-40-saga-e-bdd-integrado.md#cenários-bdd-obrigatórios). O primeiro, "Concluir fluxo de OS com pagamento aprovado", é o obrigatório pelo enunciado.

## Passos

1. `Dockerfile` do OS e de Operações.
2. Compose integrado com broker único e script de execução.
3. Steps do cenário obrigatório.
4. Demais cenários e execução no CI.
5. Evidência com trace correlacionado.

## Evidências

- Arquivos Gherkin executáveis e relatório do CI.
- Trace correlacionado do fluxo completo e de uma compensação.

Registradas em 2026-10-13:

| Evidência | Onde | Resultado |
|---|---|---|
| Cenários Gherkin executáveis | `tech-challenge-os/e2e/OficinaMecanica.E2E/Features/SagaOrdemServico.feature` | 5 cenários, o obrigatório marcado `@obrigatorio` |
| Execução local | `e2e/executar.sh` | 5 de 5 aprovados, duas vezes seguidas sobre o mesmo ambiente |
| Execução no CI | [GitHub Actions](https://github.com/LucazDenadai/tech-challenge-os/actions/runs/38448697151) | 5 de 5 aprovados com os três repositórios; `bdd.trx` como artefato |
