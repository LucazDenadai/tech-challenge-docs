# CARD-40c — Ambiente integrado e BDD ponta a ponta

**Tipo:** Testes / Integração distribuída
**Status:** To Do — decisões de implementação a definir no início do card
**Depende de:** CARD-40a, CARD-40b
**Bloqueia:** CARD-42, CARD-43
**Repositório alvo:** `tech-challenge-os`, pasta `e2e/` (decidido pelo time em 2026-10-13)
**Decisão arquitetural:** [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md); [ADR-019](../../adr/ADR-019-integracao-mercado-pago-fase4.md) (Billing em modo `Simulado`)

---

## Contexto

O enunciado exige pelo menos um fluxo completo em BDD. Hoje cada serviço tem o próprio Compose com o próprio RabbitMQ, e não existe ferramenta de BDD nos repositórios. Este card sobe os três serviços com um broker só e automatiza os cenários Gherkin do [CARD-40](CARD-40-saga-e-bdd-integrado.md#cenários-bdd-obrigatórios).

**Por que no repositório do OS:** o OS é o orquestrador e dono da Saga, então o teste ponta a ponta mora com ele. Um repositório só para e2e separaria melhor as responsabilidades, mas criaria um sexto repositório com CI e índice próprios. O `tech-challenge-docs` não tem código nem CI de .NET.

## Escopo

- Compose em `e2e/` com OS, Billing (`MercadoPago:Modo=Simulado`), Operações, um RabbitMQ, os bancos de cada serviço e DynamoDB Local.
- Projeto BDD em .NET (ferramenta definida no início do card) com o cenário obrigatório e os demais cenários do CARD-40.
- O pagamento é confirmado pela rota do simulador de Billing, como faria o webhook.
- Execução no CI do OS com dependências isoladas; como os serviços irmãos entram no CI fica definido aqui ou no CARD-41.

## Critérios de aceite

- [ ] Fluxo completo executável: abrir OS → diagnosticar → gerar/aprovar orçamento → reservar estoque por filial → cobrar/confirmar → executar → concluir → status e histórico da OS.
- [ ] Cenários Gherkin automatizados rodam em CI com broker e bancos isolados.
- [ ] Logs/traces identificam a Saga completa; dados pessoais e segredos não aparecem na evidência.
- [ ] O BDD não depende de ambiente cloud pago nem do sandbox do Mercado Pago.

## Cenários de aceite (Gherkin)

Os cenários são os do [CARD-40](CARD-40-saga-e-bdd-integrado.md#cenários-bdd-obrigatórios). O primeiro, "Concluir fluxo de OS com pagamento aprovado", é o obrigatório pelo enunciado.

## Passos

1. Definir a ferramenta de BDD e como o CI obtém as imagens dos serviços irmãos.
2. Compose integrado com broker único.
3. Steps do cenário obrigatório.
4. Demais cenários e execução no CI.
5. Evidência com trace correlacionado.

## Evidências

- Arquivos Gherkin executáveis e relatório do CI.
- Trace correlacionado do fluxo completo e de uma compensação.
