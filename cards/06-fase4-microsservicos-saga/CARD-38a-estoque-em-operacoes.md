# CARD-38a — Estoque como capacidade do serviço Operações

**Tipo:** Implementação / Domínio
**Status:** Implementado — branch `feat/card-38a-estoque-operacoes` em `tech-challenge-operacoes`, testes verificados localmente em 2026-10-08; aguardando PR e CI (CARD-41)
**Depende de:** CARD-35, CARD-36
**Bloqueia:** CARD-38b, CARD-40
**Repositório alvo:** `tech-challenge-operacoes`
**Decisão arquitetural:** ADR-004 permanece válido quanto à fonte única da verdade; ADR-014 define o serviço owner; [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md) define PostgreSQL

---

## Contexto

O Estoque existente será incorporado ao serviço Operações junto à capacidade de execução, não copiado para os outros serviços. Migração deve preservar catálogo, saldo, movimentações e histórico útil, bem como permitir a validação/reserva necessária ao Saga.

## Mapeamento da Fase 3 (2026-10-08)

| Fase 3 (`Tech-challenge`) | Fase 4 (Operações) |
|---|---|
| `Peca` com `Valor` e `QuantidadeEstoque` global | `Peca` só com catálogo e preço de tabela; saldo sai para `SaldoEstoque` por `(filialId, pecaId)` |
| `MovimentacaoEstoque` com `Entrada`/`Saida` e `OsId` | Mesma ideia, com filial, reserva, correlação e os tipos do ciclo de reserva |
| `BaixarEstoqueUseCase` (baixa direta por `OsFinalizadaEvent`, idempotente por `OsId`) | Reserva antes do pagamento, consumo na conclusão e liberação na compensação (ADR-017) |
| `POST /estoque/disponibilidade` | Mantido como consulta de disponibilidade por filial |
| `/pecas` (CRUD) | Mantido, sob `/operacoes` |
| `Servico` e `/servicos` no Atendimento (CARD-06d) | Movidos para Operações (emenda "Catálogo de serviços" do ADR-015) |
| `FalhaProcessamento`, `/estoque/falhas` e fault consumer do MassTransit (ADR-002) | Não portados: DLQ com motivo (ADR-018) |

## Decisões de implementação (2026-10-08)

- **Camadas:** Domain, Application e Infrastructure com a pasta `Estoque/` (e `Catalogo/` para peças e serviços), mais o projeto API com `Program.cs`, health/readiness e Swagger. Execução entra no CARD-38b nas mesmas camadas.
- **Catálogo:** `Peca` (`Codigo` único, `Nome`, `Descricao`, `PrecoTabela`, `Ativo`) e `Servico` (`Nome`, `Descricao`, `Preco`, `TempoConclusaoMinutos`, `Ativo`). Preço é do catálogo, igual para todas as filiais; moeda `BRL`. Desativar não apaga: o histórico de movimentação continua referenciando o item.
- **Filial:** `FilialEstoque` (`Id` = `filialId` do OS, `Codigo`, `Ativo`). Saldo, reserva e movimentação exigem filial ativa nessa tabela.
- **Saldo:** `SaldoEstoque` com chave `(FilialId, PecaId)`, `QuantidadeDisponivel` e `QuantidadeReservada`, nunca negativos. Concorrência otimista pelo `xmin` do PostgreSQL: duas reservas da mesma peça não passam do saldo.
- **Reserva:** `Reserva` (`Id` = `reservationId`, `OsId`, `FilialId`, `CorrelationId`, `IdempotencyKey` único, `Status` `Ativa`/`Consumida`/`Liberada`) com itens (`PecaId`, `Quantidade`, `QuantidadeConsumida`, `QuantidadeLiberada`). É tudo ou nada: se uma peça falta, nenhum saldo muda e a rejeição lista os `unavailablePecaIds`.
- **Consumo e liberação:** a conclusão ou falha da execução (CARD-38b) consome a quantidade usada; a liberação devolve ao disponível só o que não foi consumido. Liberar uma reserva já liberada ou consumida não muda saldo e devolve o mesmo resultado.
- **Movimentação:** tipos `Entrada`, `Ajuste`, `Reserva`, `Consumo` e `Liberacao`, com `FilialId`, `PecaId`, `Quantidade`, `Motivo`, `OcorridoEm` e, quando vier da Saga, `OsId`, `ReservaId` e `CorrelationId`.
- **Idempotência:** inbox por `messageId` e `IdempotencyKey` único na reserva. Comando repetido devolve o resultado já gravado.
- **Outbox:** `OutboxMensagens` gravada na mesma transação do efeito; um dispatcher publica nas exchanges do ADR-018 e marca o envio. Publicação repetida é segura porque o consumidor deduplica por `messageId`.
- **API:** tudo sob `/operacoes/*`. Leitura de catálogo e saldo para funcionários; escrita de catálogo, entrada e ajuste de saldo só para `Admin`.
- **Seed:** catálogo de peças e serviços, a `FILIAL-DEMO` com o `Id` fixo e saldos dessa filial. Sem dados da Fase 3.
- **OS:** o seed do OS passa a criar a `FILIAL-DEMO` com o mesmo `Id` fixo. É uma mudança pequena em `tech-challenge-os`, em PR próprio junto com este card.

## Implementação (2026-10-08)

- **Repositório:** `tech-challenge-operacoes`, em 7 commits, um por passo: solução, domínio, casos de uso, persistência, mensageria, API e README.
- **Saga:** o consumidor processa `InventoryReservationRequested` e `InventoryReleaseRequested`. Inbox, efeito e evento na outbox ficam na mesma transação do PostgreSQL. Um teste força falha na outbox e confere que nem a inbox nem a reserva ficam gravadas.
- **Despachante da outbox:** publica com confirmação do broker e depois marca a mensagem como publicada. A entrega é ao menos uma vez; o OS deduplica por `messageId`. O `traceparent` do comando segue no evento, então comando e resposta ficam no mesmo trace.
- **Concorrência:** o `xmin` do saldo detecta duas reservas da mesma peça lidas na mesma versão. A segunda recebe conflito, o consumidor refaz a mensagem com o saldo relido e a reserva é recusada se não houver saldo. Na API, o conflito devolve `409`.
- **Conclusão da execução:** consome o usado e devolve a sobra ao disponível, porque a Saga termina em `Completed` sem pedir liberação. Na falha, consome o usado e mantém o restante reservado até o `InventoryReleaseRequested` (ADR-017).
- **Execução (CARD-38b):** a porta `IEstoqueParaExecucao` é a única entrada da Execução no Estoque. Repetir a chamada não consome de novo, porque o agregado de execução fica no DynamoDB e não há transação entre os dois stores (ADR-016).
- **API:** tudo sob `/operacoes/*`. Leitura para funcionários, escrita só para `Admin`, movimentações para `Admin` e `Atendente`. Token de cliente recebe `403`. Reserva, consumo e liberação não têm rota.
- **OS:** a `FILIAL-DEMO` passou a ter `Id` fixo no seed do OS, na branch `feat/filial-demo-id-fixo` de `tech-challenge-os` (151 unitários e 33 de integração aprovados).

### Pendências

- **Recusa repetida:** a recusa de reserva não é gravada. Uma nova mensagem com a mesma `idempotencyKey`, depois de uma recusa, reavalia o saldo e pode ser aceita. A mesma mensagem reentregue continua deduplicada pela inbox.
- **Réplicas do despachante:** com mais de uma réplica, duas podem publicar o mesmo evento. O consumidor deduplica, mas o volume duplicado cresce. Travar a leitura da outbox (`FOR UPDATE SKIP LOCKED`) fica para o CARD-41, junto com o número de réplicas.
- **Cópia da spec:** `contratos/asyncapi-saga-os.yaml` é sincronizada à mão, como no OS. A checagem de divergência no CI fica no CARD-41.
- **Postman:** a collection de Operações fica para o CARD-43, junto com a do fluxo completo.

## Critérios de aceite

- [x] Operações é a única API que cria/altera catálogo, saldo e movimentações.
- [x] Catálogo de serviços (mão de obra) é recriado em Operações (código extraído do Atendimento, dados via seed), que passa a ser a única fonte de preços de peças e serviços (emenda do ADR-015).
- [x] Catálogo, saldo, reserva e movimentação são persistidos somente no PostgreSQL dedicado de Operações.
- [x] A política de disponibilidade, reserva, baixa e liberação em compensação está alinhada ao CARD-36.
- [x] Cada movimentação tem referência de negócio, motivo, data e correlação suficientes para auditoria.
- [x] Comandos repetidos com a mesma chave idempotente não duplicam movimentos.
- [x] Banco de Operações começa vazio, com migrations próprias e seed de catálogo e saldos por filial; não há migração de dados da Fase 3.
- [x] Testes cobrem saldo insuficiente, concorrência relevante, reserva/release e duplicidade.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Manter o estoque sob ownership exclusivo de Operações

  Cenário: Consultar disponibilidade para a OS
    Dado que existem peças e saldos cadastrados em Operações
    Quando um consumidor consulta disponibilidade usando o contrato publicado
    Então a resposta contém disponibilidade atual
    E nenhum dado de saldo é lido de outro serviço

  Cenário: Liberar reserva como compensação
    Dado que uma peça foi reservada para uma Saga ainda não concluída
    E uma etapa posterior falhou
    Quando a compensação de liberação é processada
    Então a reserva é liberada uma única vez
    E a movimentação fica auditável pela correlação da Saga
```

## Passos

1. Mapear modelo e regras de estoque atuais, incluindo eventos e endpoints.
2. Migrar domínio, persistência e API para o módulo de Estoque dentro de Operações.
3. Implementar comandos idempotentes e mecanismo de reserva/liberação conforme decisão de Saga.
4. Validar o seed e o fluxo de reserva antes do deploy. Não há migração de dados da Fase 3 (emenda do ADR-015).

## Evidências

- Testes de domínio/integração, relatório de reconciliação e OpenAPI.
- Exemplo de reserva e liberação correlacionadas sem duplicidade.

Registradas em 2026-10-08, branch `feat/card-38a-estoque-operacoes`:

| Evidência | Onde | Resultado |
|---|---|---|
| OpenAPI gerado | `docs/openapi/operacoes-v1.json` | 10 rotas e 16 operações sob `/operacoes/*`, sem rota de reserva |
| Testes de domínio, casos de uso e contrato | `tests/OficinaMecanica.Operacoes.UnitTests` | 84 aprovados. O contrato é testado nos dois sentidos: comandos recebidos e eventos publicados contra o JSON Schema da spec |
| Testes de persistência, mensageria e API (PostgreSQL e RabbitMQ via Testcontainers) | `tests/OficinaMecanica.Operacoes.IntegrationTests` | 34 aprovados |
| Reserva e liberação correlacionadas sem duplicidade | `ReservaELiberacao_DevolveSaldoEPublicaInventoryReleased`, `PedidoReentregue_UmaReservaEUmEvento`, `CicloReservaFalhaLiberacao_PersisteSaldosEMovimentacoesCorrelacionadas` | Movimentações com `osId`, `reservaId` e `correlationId`; reentrega gera uma reserva e um evento |
| Relatório de reconciliação | — | Não se aplica: era da migração de dados da Fase 3, que a emenda "Dados da Fase 3 e usuários" do ADR-015 retirou |