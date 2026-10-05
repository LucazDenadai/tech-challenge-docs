# ADR-018 — Contratos assíncronos em AsyncAPI no repositório de documentação

**Status:** Aceito
**Data:** 2026-10-05
**Autores:** Time Tech Challenge — Fase 4
**Relaciona-se a:** [ADR-001](ADR-001-arquitetura-microservicos-mensageria.md), [ADR-015](ADR-015-ownership-e-infraestrutura-fase4.md), [ADR-017](ADR-017-saga-orquestrada-os-fase4.md), [CARD-36](../cards/06-fase4-microsservicos-saga/CARD-36-contratos-e-desenho-saga.md)

---

## Contexto

O ADR-017 definiu a Saga orquestrada pelo OS e o catálogo de comandos/eventos entre OS, Billing e Operações. O catálogo existe apenas como tabela em Markdown, e o passo 3 do CARD-36 exige congelar envelope e contratos versionados antes de implementar os CARD-37 a CARD-40. Os três serviços ficam em repositórios separados (ADR-015), então os contratos precisam de uma fonte única que nenhum deles possua sozinho.

As APIs REST já têm formato decidido nos cards de serviço: Swagger/OpenAPI gerado por cada serviço. Falta decidir o formato dos contratos assíncronos.

---

## Decisão

- Os contratos assíncronos da Saga são descritos em **AsyncAPI 3.0**, no arquivo [`contratos/asyncapi-saga-os.yaml`](../contratos/asyncapi-saga-os.yaml) deste repositório.
- O arquivo contém o envelope comum (ADR-017), um canal por mensagem versionada, a operação `send` do produtor e o payload em JSON Schema. O consumidor de cada canal é informado na descrição do canal.
- Cada serviço implementa seus próprios DTOs conforme a spec. Não há pacote de código compartilhado entre serviços.
- O broker continua RabbitMQ (ADR-001). Convenções:
  - canal `saga-os.<mensagem-em-kebab>.v<N>` = exchange `fanout` durável com esse nome;
  - cada consumidor declara a fila `<servico>.<canal>` e a DLQ `<servico>.<canal>.dlq`.
- Mudança compatível (campo opcional novo) mantém a versão. Mudança incompatível cria `.v2` em canal próprio, e a versão anterior continua publicada até todos os consumidores migrarem.
- Toda alteração no arquivo passa por PR neste repositório e é validada com `asyncapi validate` (CLI oficial, via Docker) antes do merge.
- REST continua documentado pelo OpenAPI gerado em cada serviço.

---

## Alternativas consideradas

### Somente JSON Schema por mensagem

**Não escolhida:** valida payloads, mas não descreve canal, produtor nem consumidor; essa informação continuaria só na tabela do ADR-017.

### Pacote C# compartilhado com os records das mensagens

**Rejeitada:** cria um artefato de código comum entre serviços que devem ser independentes e acopla o deploy à versão do pacote.

### Repositório próprio de contratos ou contratos no repositório do produtor

**Não escolhida:** um repositório novo adiciona outro pipeline sem necessidade; contratos espalhados pelos produtores obrigam cada consumidor a procurar em três lugares. Este repositório já é a fonte da verdade entre os repositórios do projeto.

---

## Consequências

### Positivas

- Contrato único, legível por máquina e revisável em PR antes de qualquer implementação.
- Serviços continuam independentes: cada um gera ou escreve seus DTOs a partir da spec.
- A documentação navegável pode ser gerada pelo CLI do AsyncAPI para a entrega final (CARD-43).

### Negativas e riscos

| Risco | Mitigação |
|---|---|
| DTOs dos serviços divergirem da spec | Testes de contrato em cada serviço validam mensagens produzidas/consumidas contra o JSON Schema da spec (CARD-37 a CARD-39). |
| Spec e ADR-017 ficarem desalinhados | A spec é a fonte dos campos; o ADR-017 mantém fluxo, estados e compensações. Alterar o catálogo exige atualizar os dois no mesmo PR. |
| Validação depender de Docker localmente | O CLI também roda em CI; a validação local é conveniência. |

---

## Referências

- [AsyncAPI 3.0 — especificação](https://www.asyncapi.com/docs/reference/specification/v3.0.0)
- [AsyncAPI CLI](https://www.asyncapi.com/docs/tools/cli)
- [ADR-017 — Saga orquestrada pelo serviço OS](ADR-017-saga-orquestrada-os-fase4.md)
- [CARD-36 — Contratos e desenho da Saga](../cards/06-fase4-microsservicos-saga/CARD-36-contratos-e-desenho-saga.md)
