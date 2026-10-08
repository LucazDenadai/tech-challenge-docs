# CARD-38a — Estoque como capacidade do serviço Operações

**Tipo:** Implementação / Domínio
**Status:** To Do
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

## Critérios de aceite

- [ ] Operações é a única API que cria/altera catálogo, saldo e movimentações.
- [ ] Catálogo de serviços (mão de obra) é recriado em Operações (código extraído do Atendimento, dados via seed), que passa a ser a única fonte de preços de peças e serviços (emenda do ADR-015).
- [ ] Catálogo, saldo, reserva e movimentação são persistidos somente no PostgreSQL dedicado de Operações.
- [ ] A política de disponibilidade, reserva, baixa e liberação em compensação está alinhada ao CARD-36.
- [ ] Cada movimentação tem referência de negócio, motivo, data e correlação suficientes para auditoria.
- [ ] Comandos repetidos com a mesma chave idempotente não duplicam movimentos.
- [ ] Banco de Operações começa vazio, com migrations próprias e seed de catálogo e saldos por filial; não há migração de dados da Fase 3.
- [ ] Testes cobrem saldo insuficiente, concorrência relevante, reserva/release e duplicidade.

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