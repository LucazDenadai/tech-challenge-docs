# CARD-38 — Serviço Operações (Estoque + Execução)

**Tipo:** Implementação / Microsserviço
**Status:** Em andamento — CARD-38a concluído e CARD-38b implementado; falta deploy independente e CI (CARD-41) e o cancelamento de execução (CARD-40)
**Depende de:** CARD-34, CARD-35, CARD-36
**Bloqueia:** CARD-40, CARD-41, CARD-43
**Repositório alvo:** `tech-challenge-operacoes` (ADR-015)
**Decisão arquitetural:** [ADR-014](../../adr/ADR-014-limites-microsservicos-fase4.md), [ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md), [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md), [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md), [ADR-018](../../adr/ADR-018-contratos-assincronos-asyncapi-fase4.md)

---

## Contexto

ADR-014 mantém Estoque e Execução no mesmo microsserviço Operações para atender ao mínimo de três serviços sem criar um serviço adicional. Operações é owner exclusivo do catálogo/saldo/movimentações e da fila/estado de execução. As capacidades permanecem módulos internos distintos para evitar que a junção se transforme em acoplamento indiscriminado.

## Decisões de escopo (2026-10-08)

- **Repositório:** `tech-challenge-operacoes`, público, criado como o do OS. Um único executável com dois módulos internos, Estoque e Execução, em projetos e namespaces `OficinaMecanica.Operacoes.*` (padrão `OficinaMecanica.<Contexto>.*`).
- **Módulos:** cada módulo tem pastas próprias em Domain, Application e Infrastructure. Execução usa Estoque só pela porta de aplicação do módulo (reservar, consumir, liberar), nunca pelas tabelas. Isso mantém os dois agregados distintos (ADR-014) e cada um no seu store (ADR-016).
- **Fronteira com o CARD-40:** Operações é participante da Saga, não orquestrador. Os handlers dos comandos que Operações recebe (diagnóstico, reserva, início, liberação) aplicam o próprio efeito e publicam o resultado pela outbox neste card. O CARD-40 implementa a orquestração no OS e o BDD integrado. É diferente do OS (CARD-37b), em que o efeito de negócio foi para o CARD-40 porque o ADR-017 só permite mudar o status pela transição da Saga.
- **Mensageria:** mesma abordagem do OS: `RabbitMQ.Client` direto, filas `operacoes.<canal>` e DLQs `operacoes.<canal>.dlq` (ADR-018), inbox por `messageId`, 3 retentativas (1 s, 5 s, 10 s) antes da DLQ com motivo. Sem tabela de falhas: o [ADR-002](../../adr/ADR-002-observabilidade-falhas-tabela-banco.md) foi substituído pelo ADR-018.
- **Autenticação:** valida o JWT emitido pelo OS (mesmo issuer e chave); não emite tokens (emenda "Dados da Fase 3 e usuários" do ADR-015).
- **Filial:** Operações guarda só as filiais onde opera estoque, com o `Id` do OS. A `FILIAL-DEMO` tem `Id` fixo nos dois seeds (emenda "Filiais em Operações" do ADR-015).
- **Ordem:** 38a (Estoque e catálogo no PostgreSQL, com o projeto executável) e depois 38b (Execução no DynamoDB). Cada subcard em PR próprio, com commits por passo.

## Escopo

- Repositório, deploy e infraestrutura exclusivos de Operações; PostgreSQL dedicado para estoque e DynamoDB dedicado para execução.
- Manter Estoque como única fonte da verdade para peças e saldos.
- Manter o catálogo de serviços (mão de obra) e os preços de tabela de peças e serviços (emenda do ADR-015).
- Receber solicitação de diagnóstico na abertura da OS e publicar os itens necessários ao orçamento.
- Receber solicitação aprovada para iniciar trabalho e controlar fila, reparo e conclusão.
- Validar disponibilidade/reservar ou baixar peças segundo contratos e Saga do CARD-36.
- Publicar para OS/Saga os eventos de diagnóstico, reserva, início, conclusão e falha definidos no [AsyncAPI](../../contratos/asyncapi-saga-os.yaml) (ADR-018). O progresso do reparo é consultado na API de Operações, sem evento para o OS.
- Expor APIs para catálogo/saldo e operações administrativas necessárias, com Swagger.

## Fora de escopo

- Manter cópia gerenciável do saldo de peças em OS ou Billing.
- Aprovar orçamento ou registrar pagamento.
- Alterar diretamente estado de OS no banco de OS.

## Critérios de aceite

- [ ] Operações pode ser compilado, testado, implantado e escalado sem publicar/reiniciar OS ou Billing. *(Compila e testa sozinho; Dockerfile, manifests e workflow ficam no CARD-41.)*
- [x] Operações possui a instância PostgreSQL e a tabela DynamoDB exclusivas definidas em ADR-016; Estoque e Execução permanecem módulos internos distintos.
- [x] API/eventos suportam disponibilidade, fila de execução, diagnóstico, progresso, conclusão e falha.
- [x] Operações publica eventos contratados, com correlação e versionamento, sem gravar no banco OS/Billing.
- [x] Estados de Execução e Movimentações de Estoque são distintos e têm regras de domínio próprias.
- [x] Reprocessamento de comando/evento não duplica movimentação, reserva ou item de execução.
- [ ] Testes cobrem estoque insuficiente, execução cancelada/finalizada e falha de persistência/mensagem. *(Cobertos, menos execução cancelada, que depende do CARD-40.)*
- [x] A API de fila/execução lê e escreve o agregado de execução no DynamoDB; saldo, reservas e movimentações permanecem exclusivamente no PostgreSQL.
- [x] A configuração DynamoDB Local permite executar testes sem credenciais AWS ou chamadas ao endpoint cloud.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Operar estoque e execução em um serviço com ownership único

  Cenário: Enfileirar OS aprovada
    Dado que Operações recebeu uma solicitação válida e correlacionada
    E os itens necessários estão disponíveis conforme a política aprovada
    Quando a solicitação é aceita
    Então uma execução é registrada na fila de Operações
    E o estado de execução fica consultável
    E nenhum banco externo é alterado diretamente

  Cenário: Não iniciar execução quando faltar peça
    Dado que ao menos uma peça necessária não está disponível
    Quando Operações avalia a solicitação
    Então a execução não avança para o estado iniciado
    E um resultado de rejeição é publicado para a Saga
    E nenhum saldo fica parcialmente alterado sem compensação registrada

  Cenário: Concluir reparo e informar OS
    Dado que uma execução está em reparo
    Quando o responsável registra a conclusão
    Então Operações persiste a conclusão em seu banco
    E publica evento de conclusão com o identificador da OS e correlação
```

## Subcards

- [CARD-38a — Estoque como capacidade de Operações](CARD-38a-estoque-em-operacoes.md)
- [CARD-38b — Fila e ciclo de execução](CARD-38b-fila-execucao.md)

## Evidências

- Link do repositório, testes, OpenAPI, workflow e manifests.
- Prova de que o banco contém apenas dados sob ownership de Operações.
- Trace/eventos de um fluxo de execução completo.

Registradas até 2026-10-08:

- Repositório: [tech-challenge-operacoes](https://github.com/LucazDenadai/tech-challenge-operacoes), PR #1 (38a) e branch do 38b.
- Testes e OpenAPI: tabelas de evidências do [CARD-38a](CARD-38a-estoque-em-operacoes.md#evidências) e do [CARD-38b](CARD-38b-fila-execucao.md#evidências).
- Banco só com dados de Operações: teste `Migrations_CriamSomenteCatalogoEEstoque` e tabela DynamoDB própria.
- Fluxo de execução completo: `FluxoDaExecucao_DiagnosticoReservaInicioEConclusao`, com eventos correlacionados e trace contínuo.
- Workflow e manifests: pendentes (CARD-41).