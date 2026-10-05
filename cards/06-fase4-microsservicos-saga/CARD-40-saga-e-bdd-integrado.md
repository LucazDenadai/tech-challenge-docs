# CARD-40 — Implementar Saga e validar fluxo integrado com BDD

**Tipo:** Integração distribuída / Testes
**Status:** To Do
**Depende de:** CARD-36, CARD-37, CARD-38, CARD-39
**Bloqueia:** CARD-42, CARD-43
**Repositórios:** OS, Operações, Billing e `tech-challenge-docs`
**Decisão arquitetural:** ADR de estratégia da Saga aprovado no CARD-36

---

## Contexto

O enunciado requer coordenação distribuída entre abertura de OS, orçamento, aprovação, pagamento e execução, com rollback/compensação em caso de falha. Transação local, retry de RabbitMQ e DLQ não substituem uma Saga. A implementação segue a estratégia e máquina de estados decididas em CARD-36, persiste progresso e tolera redelivery.

Este card entrega o cenário BDD ponta a ponta obrigatório. O texto Gherkin deste card é a especificação; ao menos um cenário de sucesso e cenários representativos de falha devem ser executáveis no pipeline.

## Escopo

- Implementar coordenação selecionada no ADR, estados persistidos e transições válidas.
- Garantir publicação confiável de comandos/eventos com outbox/inbox ou estratégia aprovada equivalente.
- Propagar Saga/correlation ID através de API, broker, Billing, Mercado Pago e Operações.
- Implementar idempotência, retry limitado, timeout, recuperação após restart e tratamento de mensagem inválida.
- Executar compensações financeiras/operacionais quando possível e fornecer recuperação manual auditável quando não for possível.
- Automatizar BDD integrado sem depender de ambiente cloud pago em cada execução.

## Critérios de aceite

- [ ] Fluxo completo documentado e executável: abrir OS → gerar orçamento → aguardar aprovação → pagar/confirmar → autorizar execução → concluir → atualizar estado/histórico da OS.
- [ ] Estado da Saga sobrevive a restart e pode ser consultado sem acesso cruzado aos bancos dos serviços.
- [ ] Toda etapa tem timeout/retry explícitos; redelivery não duplica pagamento, reserva, execução ou transição de OS.
- [ ] Falha em cada fronteira crítica tem teste para rejeição, retry ou compensação apropriada.
- [ ] Pagamento aprovado não é marcado como desfeito até confirmação de estorno; estado pendente de estorno permanece explícito.
- [ ] Falha de reserva/execução após aprovação aciona a compensação definida (incluindo estorno ou recuperação manual conforme decisão do CARD-36).
- [ ] Eventos publicados após persistência local seguem a garantia transacional decidida e não são perdidos silenciosamente.
- [ ] Cenários Gherkin automatizados rodam em CI com dependências isoladas, incluindo broker e bancos necessários.
- [ ] Logs/traces permitem identificar a Saga completa; dados pessoais e segredos não aparecem na evidência.

## Cenários BDD obrigatórios

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
    Então a Saga termina no estado cancelado ou rejeitado definido no ADR
    E nenhuma cobrança é criada
    E Operações não inicia a execução
    E OS recebe o resultado por contrato, sem acesso ao banco de Billing

  Cenário: Compensar falha operacional após pagamento aprovado
    Dado que o pagamento foi confirmado
    E Operações rejeita a reserva ou não consegue iniciar a execução
    Quando a Saga processa a falha definitiva
    Então a compensação definida é solicitada ao Billing e a Operações
    E cada resultado é persistido e correlacionado
    E a Saga só termina como compensada após confirmação dos efeitos compensatórios

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