# CARD-38 — Serviço Operações (Estoque + Execução)

**Tipo:** Implementação / Microsserviço
**Status:** To Do
**Depende de:** CARD-34, CARD-35, CARD-36
**Bloqueia:** CARD-40, CARD-41, CARD-43
**Repositório alvo:** Repositório exclusivo de Operações (nome definido no CARD-34)
**Decisão arquitetural:** [ADR-014](../../adr/ADR-014-limites-microsservicos-fase4.md), [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md)

---

## Contexto

ADR-014 mantém Estoque e Execução no mesmo microsserviço Operações para atender ao mínimo de três serviços sem criar um serviço adicional. Operações é owner exclusivo do catálogo/saldo/movimentações e da fila/estado de execução. As capacidades permanecem módulos internos distintos para evitar que a junção se transforme em acoplamento indiscriminado.

## Escopo

- Repositório, deploy e infraestrutura exclusivos de Operações; PostgreSQL dedicado para estoque e DynamoDB dedicado para execução.
- Manter Estoque como única fonte da verdade para peças e saldos.
- Receber solicitação aprovada para iniciar trabalho e controlar fila, diagnóstico, reparo e conclusão.
- Validar disponibilidade/reservar ou baixar peças segundo contratos e Saga do CARD-36.
- Publicar eventos de progresso, conclusão e falha para OS/Saga.
- Expor APIs para catálogo/saldo e operações administrativas necessárias, com Swagger.

## Fora de escopo

- Manter cópia gerenciável do saldo de peças em OS ou Billing.
- Aprovar orçamento ou registrar pagamento.
- Alterar diretamente estado de OS no banco de OS.

## Critérios de aceite

- [ ] Operações pode ser compilado, testado, implantado e escalado sem publicar/reiniciar OS ou Billing.
- [ ] Operações possui a instância PostgreSQL e a tabela DynamoDB exclusivas definidas em ADR-016; Estoque e Execução permanecem módulos internos distintos.
- [ ] API/eventos suportam disponibilidade, fila de execução, diagnóstico, progresso, conclusão e falha.
- [ ] Operações publica eventos contratados, com correlação e versionamento, sem gravar no banco OS/Billing.
- [ ] Estados de Execução e Movimentações de Estoque são distintos e têm regras de domínio próprias.
- [ ] Reprocessamento de comando/evento não duplica movimentação, reserva ou item de execução.
- [ ] Testes cobrem estoque insuficiente, execução cancelada/finalizada e falha de persistência/mensagem.
- [ ] A API de fila/execução lê e escreve o agregado de execução no DynamoDB; saldo, reservas e movimentações permanecem exclusivamente no PostgreSQL.
- [ ] A configuração DynamoDB Local permite executar testes sem credenciais AWS ou chamadas ao endpoint cloud.

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