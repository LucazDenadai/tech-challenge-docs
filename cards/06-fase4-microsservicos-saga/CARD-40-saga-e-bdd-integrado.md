# CARD-40 — Implementar Saga e validar fluxo integrado com BDD

**Tipo:** Integração distribuída / Testes
**Status:** To Do
**Depende de:** CARD-36, CARD-37, CARD-38, CARD-39
**Bloqueia:** CARD-42, CARD-43
**Repositórios:** OS, Operações, Billing e `tech-challenge-docs`
**Decisão arquitetural:** [ADR-017 — Saga orquestrada pelo serviço OS](../../adr/ADR-017-saga-orquestrada-os-fase4.md)

---

## Contexto

O enunciado requer coordenação distribuída entre abertura de OS, orçamento, aprovação, pagamento e execução, com compensação em caso de falha. Transação local, retry de RabbitMQ e DLQ não substituem uma Saga. ADR-017 define o orchestrator no OS, confirmado pelo time.

Este card entregará o cenário BDD ponta a ponta obrigatório. Os cenários Gherkin abaixo são a especificação de comportamento, com estratégia e contratos definidos no CARD-36/ADR-017.

## Escopo

- Implementar a estratégia e os estados persistidos selecionados e confirmados no CARD-36.
- Garantir publicação confiável de comandos/eventos com outbox/inbox ou estratégia aprovada equivalente.
- Propagar Saga/correlation ID através de API, broker, Billing, Mercado Pago e Operações.
- Implementar idempotência, retry limitado, timeout, recuperação após restart e tratamento de mensagem inválida.
- Executar compensações financeiras/operacionais quando possível e fornecer recuperação manual auditável quando não for possível.
- Automatizar BDD integrado sem depender de ambiente cloud pago em cada execução.
- Processar os eventos já registrados na inbox `InboxMensagens` do OS pelo CARD-37b (validação, deduplicação e DLQ já existem) e aplicar o efeito de negócio.
- Integrar os handlers participantes de Operações já entregues no CARD-38 (efeito local e resultado pela outbox). Este card implementa a orquestração no OS e o BDD ponta a ponta, sem reimplementar o efeito de Operações.
- Levar o cancelamento manual da OS (`POST /os/ordens-servico/{id}/cancelamento`, CARD-37b) pela Saga: hoje, a partir de `AguardandoPagamento` ou `EmExecucao`, ele só muda o status, sem liberar reserva nem estornar.

## Critérios de aceite

- [x] Estratégia Saga foi confirmada pelo time no CARD-36 antes da implementação deste card.
- [ ] Fluxo completo documentado e executável: abrir OS → diagnosticar em Operações → gerar/aprovar orçamento → reservar estoque por filial → cobrar/confirmar → autorizar execução → concluir → atualizar estado/histórico da OS.
- [ ] Estado da Saga sobrevive a restart e pode ser consultado sem acesso cruzado aos bancos dos serviços.
- [ ] Toda etapa tem timeout/retry explícitos; redelivery não duplica pagamento, reserva, execução ou transição de OS.
- [ ] Falha em cada fronteira crítica tem teste para rejeição, retry ou compensação apropriada.
- [ ] Pagamento aprovado não é marcado como desfeito até confirmação de estorno; estado pendente de estorno permanece explícito.
- [ ] Falha de reserva antes da cobrança cancela sem criar pagamento; falha após aprovação do pagamento (início rejeitado ou execução falhou) aciona liberação da reserva não consumida e estorno total conforme ADR-017.
- [ ] Cada etapa da tabela de compensações do ADR-017 tem caminho de falha implementado e teste de integração; itens marcados como evolução no Escopo do MVP do ADR-017 não são implementados.
- [ ] Resultado de pagamento desconhecido mantém Saga em reconciliação, sem marcar como recusado, cobrar de novo ou liberar reserva indevidamente.
- [ ] Eventos publicados após persistência local seguem a garantia transacional decidida e não são perdidos silenciosamente.
- [ ] Cenários Gherkin automatizados rodam em CI com dependências isoladas, incluindo broker e bancos necessários.
- [ ] Logs/traces permitem identificar a Saga completa; dados pessoais e segredos não aparecem na evidência.

## Cenários BDD obrigatórios

O PDF exige pelo menos um fluxo completo em BDD: o primeiro cenário é o obrigatório. Os demais são automatizados em BDD ou como testes de integração.

```gherkin
Funcionalidade: Coordenar uma OS distribuída por Saga

  Cenário: Concluir fluxo de OS com pagamento aprovado
    Dado que cliente e veículo estão aptos na filial selecionada
    Quando a OS é aberta e o orçamento é aprovado
    E o Mercado Pago confirma o pagamento
    E Operações inicia e conclui o reparo
    Então a Saga termina no estado concluído
    E OS apresenta o status e histórico correspondentes
    E Billing mantém o pagamento aprovado
    E Operações mantém a execução concluída e as movimentações esperadas

  Cenário: Rejeitar orçamento sem iniciar pagamento ou execução
    Dado que a OS foi aberta
    Quando o cliente rejeita o orçamento
    Então a Saga termina no estado Cancelled
    E nenhuma cobrança é criada
    E Operações não inicia a execução
    E OS recebe o resultado por contrato, sem acesso ao banco de Billing

  Cenário: Compensar falha operacional após pagamento aprovado
    Dado que o pagamento foi confirmado
    E Operações não consegue iniciar a execução ou a execução falha
    Quando a Saga processa a falha definitiva
    Então OS solicita estorno total ao Billing e liberação da reserva não consumida a Operações
    E cada resultado é persistido e correlacionado
    E a Saga só termina como compensada após confirmação dos efeitos compensatórios

  Cenário: Não cobrar quando falta estoque na filial
    Dado que o orçamento foi aprovado para uma OS da filial A
    E Operações não tem saldo suficiente para reservar os itens na filial A
    Quando a Saga recebe InventoryReservationRejected
    Então não solicita pagamento ao Mercado Pago
    E não consome estoque de outra filial
    E OS recebe o resultado de cancelamento por contrato

  Cenário: Reconciliar resultado de pagamento desconhecido
    Dado que uma tentativa idempotente foi enviada ao Mercado Pago
    E Billing não consegue consultar resultado autoritativo por timeout
    Quando OS recebe PaymentOutcomeUnknown
    Então mantém estado ReconcilingPayment
    E Billing consulta/reconcilia a mesma tentativa
    E nenhuma segunda cobrança é criada

  Cenário: Recuperar Saga interrompida após reinício
    Dado que um serviço reiniciou após persistir uma etapa da Saga
    Quando o serviço volta a processar mensagens
    Então retoma do estado persistido
    E não repete efeitos já confirmados

  Cenário: Tratar webhook duplicado
    Dado que a confirmação de pagamento já foi processada
    Quando o mesmo webhook é entregue novamente
    Então a Saga não avança duas vezes
    E nenhum evento de autorização de execução é duplicado
```

## Passos

1. Converter máquina de estados e tabela de compensações aprovada em handlers/comandos.
2. Implementar persistência de Saga e outbox/inbox ou alternativa decidida.
3. Implementar consumidores idempotentes e correlação ponta a ponta.
4. Criar suite BDD com ambientes de teste reproduzíveis e dados sintéticos.
5. Injetar falhas em cada fronteira; validar compensação, retry, restart e mensagens duplicadas.
6. Publicar execução e evidências sem expor dados reais.

## Evidências

- Arquivos Gherkin executáveis, relatório CI e cenários de falha.
- Trace correlacionado de sucesso e de compensação.
- Documento de máquina de estados, recuperação e resultados de falha.