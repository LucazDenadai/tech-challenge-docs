# CARD-36 — Contratos entre serviços e desenho da Saga

**Tipo:** Arquitetura / Integração
**Status:** Concluído — estratégia confirmada pelo time em 2026-10-05
**Depende de:** CARD-34, CARD-35
**Bloqueia:** CARD-37, CARD-38, CARD-39, CARD-40
**Repositórios:** `tech-challenge-docs` e repositórios dos três serviços
**Decisão arquitetural:** [ADR-017 — Saga orquestrada pelo serviço OS](../../adr/ADR-017-saga-orquestrada-os-fase4.md)

---

## Contexto

O fluxo obrigatório atravessa OS, Billing e Operações. Mensageria, retry e DLQ já existem no projeto, mas não constituem por si só uma Saga. ADR-017 registra a orquestração pelo OS, confirmada pelo time, com estado persistido no banco OS e comandos versionados. A coreografia foi avaliada e não escolhida.

## Escopo

Definir contratos REST/eventos, correlation/causation IDs, versionamento, ordenação e compatibilidade; máquina de estados da Saga; persistência do estado; timeout/retry/idempotência; estratégia outbox/inbox ou equivalente; política de mensagens inválidas/DLQ; eventos de aprovação e notificação Mercado Pago; e compensações para falhas de cobrança, reserva/baixa de peça e início/cancelamento da execução.

O diagnóstico ocorre em Operações logo após a abertura e define os itens do orçamento, mantendo a ordem da Fase 3. O fluxo reserva peças da filial da OS antes de criar uma cobrança. Timeout/erro técnico do provedor resulta em reconciliação, nunca em recusa presumida. Pagamento aprovado seguido de falha operacional requer estorno e liberação/ajuste de reserva como compensações separadas. A Saga só declara compensação concluída quando todos os efeitos reversíveis forem confirmados; efeitos físicos não reversíveis requerem registro real do consumo e intervenção manual quando necessário.

## Critérios de aceite

- [x] Diagrama de sequência mostra o fluxo normal da abertura à execução e conclusão.
- [x] Diagrama de estados identifica responsável por cada transição e estado persistido da Saga.
- [x] ADR compara orquestração e coreografia e apresenta uma recomendação fundamentada para avaliação.
- [x] Contratos identificam produtor, consumidores, payload versionado e política de compatibilidade.
- [x] Cada comando/evento inclui identificadores de correlação/causação e idempotência suficientes para rastrear e deduplicar o processo.
- [x] Cada etapa tem timeout/retry, estado para resultado desconhecido e destino de falha não recuperável.
- [x] Efeitos reversíveis têm compensação; efeitos físicos não reversíveis têm estado real e caminho manual auditável.
- [x] Aprovação no Billing é distinta da confirmação reconciliada do webhook Mercado Pago; duplicatas/out-of-order não repetem efeitos.
- [x] Persistência de Saga/outbox respeita ownership; não há transação distribuída nem acesso cruzado a bancos.
- [x] Diagramas Mermaid de sequência e estados foram adicionados.
- [x] O time confirma a estratégia de orquestração, coreografia ou alternativa antes de congelar contratos de implementação (orquestração pelo OS, 2026-10-05).

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Definir contratos resilientes para a Saga da OS

  Cenário: Correlacionar eventos de uma ordem
    Dado que a equipe confirmou uma estratégia Saga para implementação
    E uma OS inicia essa Saga
    Quando um serviço publica um evento de domínio
    Então o evento carrega o identificador de correlação da Saga
    E o serviço consumidor registra esse identificador junto ao resultado

  Cenário: Definir compensação para falha após aprovação de pagamento
    Dado que a equipe avaliou e confirmou o fluxo de compensação proposto
    E o pagamento foi confirmado pelo provedor
    E a etapa seguinte do fluxo falhou
    Quando a compensação é executada
    Então a ação explícita de compensação ou recuperação é iniciada
    E a ação tem estado, política de repetição e evidência de resultado

  Cenário: Não cobrar se a reserva de estoque falhar
    Dado que o orçamento foi aprovado
    E Operações não conseguiu reservar todos os itens na filial da OS
    Quando OS processa a rejeição de reserva
    Então a Saga termina sem solicitar pagamento ao Mercado Pago
    E OS registra a razão do cancelamento

  Cenário: Manter pagamento em reconciliação quando o provedor está indisponível
    Dado que Billing solicitou uma cobrança com chave idempotente
    E a chamada ou consulta ao Mercado Pago expirou sem resultado autoritativo
    Quando Billing publica PaymentOutcomeUnknown
    Então OS mantém a Saga em ReconcilingPayment
    E nenhuma recusa ou aprovação é presumida
    E Billing reconcilia a mesma tentativa sem criar cobrança duplicada

  Cenário: Compensar cobrança aprovada quando execução não pode começar
    Dado que Billing confirmou o pagamento
    E Operações rejeitou o início antes de consumir peças ou realizar trabalho
    Quando OS inicia compensação
    Então comanda liberação da reserva e estorno do pagamento
    E só marca a Saga como Compensated após confirmar ambos os resultados

  Cenário: Não reverter trabalho físico por rollback lógico
    Dado que a execução consumiu peças ou realizou trabalho antes de falhar
    Quando a Saga trata a falha
    Então Operações registra o consumo real e libera apenas itens não consumidos
    E a compensação financeira fica explícita como estorno ou ação manual
    E a Saga não afirma que o efeito físico foi desfeito
```

## Passos

1. Estudar as referências de Saga listadas no ADR-017 e comparar coreografia/orquestração para este fluxo.
2. Revisar a recomendação e os cenários com os responsáveis por OS, Billing e Operações; registrar a confirmação ou uma alternativa escolhida.
3. Após a confirmação, congelar envelope e contratos versionados (OpenAPI/AsyncAPI ou formato selecionado) para CARD-37 a CARD-39.
4. Implementar transições/timeout/retry/outbox por serviço e automatizar cenários no CARD-40; não iniciar essa implementação com o desenho ainda pendente.

## Evidências

- [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md) aceito, com estratégia, estados, contratos, retries, compensações e escopo do MVP.
- [Diagrama de sequência](../../diagramas/diagrama-sequencia-saga-fase4.md), [diagrama de estados](../../diagramas/diagrama-estados-saga-os-fase4.md) e catálogo de eventos.
- Contratos versionados e matriz etapa → falha → compensação → verificação registrados no ADR-017.