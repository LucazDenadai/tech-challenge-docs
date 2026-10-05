# CARD-38a — Estoque como capacidade do serviço Operações

**Tipo:** Implementação / Domínio
**Status:** To Do
**Depende de:** CARD-35, CARD-36
**Bloqueia:** CARD-38b, CARD-40
**Repositório alvo:** Repositório exclusivo de Operações
**Decisão arquitetural:** ADR-004 permanece válido quanto à fonte única da verdade; ADR-014 define o serviço owner; [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md) define PostgreSQL

---

## Contexto

O Estoque existente será incorporado ao serviço Operações junto à capacidade de execução, não copiado para os outros serviços. Migração deve preservar catálogo, saldo, movimentações e histórico útil, bem como permitir a validação/reserva necessária ao Saga.

## Critérios de aceite

- [ ] Operações é a única API que cria/altera catálogo, saldo e movimentações.
- [ ] Catálogo de serviços (mão de obra) do Atendimento legado é migrado para Operações, que passa a ser a única fonte de preços de peças e serviços (emenda do ADR-015).
- [ ] Catálogo, saldo, reserva e movimentação são persistidos somente no PostgreSQL dedicado de Operações.
- [ ] A política de disponibilidade, reserva, baixa e liberação em compensação está alinhada ao CARD-36.
- [ ] Cada movimentação tem referência de negócio, motivo, data e correlação suficientes para auditoria.
- [ ] Comandos repetidos com a mesma chave idempotente não duplicam movimentos.
- [ ] Migração do Estoque legado tem reconciliação de catálogo e saldos, com plano de rollback/cutover.
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
4. Validar migração e reconciliação antes do corte de tráfego.

## Evidências

- Testes de domínio/integração, relatório de reconciliação e OpenAPI.
- Exemplo de reserva e liberação correlacionadas sem duplicidade.