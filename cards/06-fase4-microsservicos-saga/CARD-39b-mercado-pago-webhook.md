# CARD-39b — Mercado Pago: cobrança e webhook

**Tipo:** Integração externa / Pagamentos
**Status:** To Do
**Depende de:** CARD-36, CARD-39a
**Bloqueia:** CARD-40
**Repositório alvo:** Repositório exclusivo de Billing
**Decisão arquitetural:** Contrato e compensação aprovados no CARD-36

---

## Contexto

Pagamento é uma fronteira assíncrona: criação de preferência/intenção não confirma liquidação e o retorno do navegador não é fonte confiável. Billing deve integrar-se à API atual do Mercado Pago, validar notificações e reconciliar o estado consultando o provedor conforme documentação oficial selecionada. A integração começa em sandbox.

## Critérios de aceite

- [ ] ADR/RFC de integração registra produto/API do Mercado Pago escolhido, autenticação, estados, idempotency key e política de retry/rate limit.
- [ ] Criação de pagamento associa identificador interno, OS, orçamento, filial e identificador do provedor sem salvar dados de cartão.
- [ ] Webhook verifica assinatura/autenticidade segundo a documentação vigente e valida o recurso consultando o provedor quando aplicável.
- [ ] A chave idempotente impede cobrança duplicada em retry da mesma solicitação.
- [ ] Webhook repetido, atrasado, inválido e de estado terminal não regressa estado nem duplica evento.
- [ ] O pagamento consultado no provedor corresponde à tentativa interna, OS, orçamento, valor e moeda antes de qualquer confirmação.
- [ ] Estado de recusa do provedor é tratado como resultado de negócio; erro HTTP, timeout ou indisponibilidade permanece resultado técnico desconhecido e segue retry/reconciliação.
- [ ] Webhook válido para pagamento desconhecido ou com associação divergente não altera estado e segue o tratamento de segurança/auditoria definido.
- [ ] Falha temporária do provedor gera retry limitado e observável; falha persistente é encaminhada para reconciliação/DLQ conforme decisão.
- [ ] Aprovação publicada só ocorre após confirmação autoritativa do pagamento.
- [ ] Caminho de cancelamento/estorno ou recuperação manual está exercitado para falha posterior na Saga.
- [ ] Testes cobrem sandbox e cenários simulados sem expor tokens em log.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Integrar pagamentos com Mercado Pago

  Cenário: Criar pagamento sem duplicar cobrança
    Dado que existe um orçamento aprovado
    Quando a mesma solicitação idempotente de pagamento é enviada duas vezes
    Então existe no máximo uma cobrança correspondente no provedor
    E Billing retorna a mesma referência de pagamento

  Cenário: Receber webhook de pagamento aprovado
    Dado que o provedor envia uma notificação válida de pagamento
    E a consulta ao provedor confirma o estado aprovado
    Quando Billing processa o webhook
    Então o estado financeiro é persistido como aprovado
    E um único evento de confirmação é publicado

  Cenário: Não aceitar webhook adulterado
    Dado que a assinatura da notificação é inválida
    Quando Billing recebe o webhook
    Então a notificação é rejeitada e auditada
    E o estado de pagamento permanece inalterado

  Cenário: Não confirmar tentativa com valor ou moeda divergentes
    Dado que um webhook autenticado referencia uma tentativa existente
    Quando Billing consulta o pagamento no Mercado Pago
    E o valor ou a moeda não correspondem ao orçamento aprovado
    Então Billing não marca o pagamento como aprovado
    E não publica evento de pagamento aprovado
    E registra a divergência sem expor dados sensíveis

  Cenário: Registrar recusa de pagamento sem liberar execução
    Dado que o Mercado Pago confirma que a tentativa foi recusada
    Quando Billing processa a notificação e reconcilia o estado
    Então a tentativa é registrada como recusada
    E a Saga recebe o evento de recusa definido no contrato
    E nenhum evento de pagamento aprovado é publicado

  Cenário: Não tratar indisponibilidade como recusa
    Dado que a consulta ao Mercado Pago falha por timeout ou indisponibilidade
    Quando Billing processa uma notificação ainda não reconciliada
    Então o pagamento permanece pendente ou com resultado desconhecido
    E Billing agenda retry/reconciliação conforme política
    E não publica evento de recusa nem de aprovação

  Cenário: Não regredir pagamento por notificação fora de ordem
    Dado que Billing já registrou o pagamento como aprovado
    Quando recebe uma notificação válida de estado pendente anterior
    Então mantém o pagamento aprovado
    E não publica uma nova transição incompatível

  Cenário: Colocar em investigação pagamento desconhecido ou associado a outra OS
    Dado que um webhook autenticado referencia um pagamento inexistente ou incompatível com a OS ou orçamento
    Quando Billing tenta reconciliar o pagamento
    Então nenhum orçamento ou pagamento é alterado para aprovado
    E o evento não é publicado para autorizar execução
    E a ocorrência é registrada para investigação conforme política de segurança

  Cenário: Não duplicar estorno após reentrega da compensação
    Dado que Billing já confirmou o estorno de uma tentativa aprovada
    Quando a mesma solicitação de compensação é recebida novamente
    Então Billing retorna o resultado já persistido
    E não solicita um segundo estorno ao Mercado Pago

  Cenário: Estornar quando uma etapa posterior exigir compensação
    Dado que o pagamento foi aprovado
    E a Saga solicitou compensação por falha posterior
    Quando o estorno é solicitado ao provedor
    Então Billing registra o resultado do estorno
    E publica o evento de compensação apenas após confirmação
```

## Passos

1. Conferir a documentação oficial do Mercado Pago e decidir produto/API compatível com o fluxo de orçamento.
2. Criar adapter de provedor atrás de porta interna e configuração por ambiente.
3. Implementar criação de pagamento e persistência da referência/idempotência.
4. Implementar webhook com verificação de autenticidade, consulta autoritativa e proteção contra replay.
5. Implementar cancelamento/estorno e tratar estados que o provedor não permita reverter.
6. Validar no sandbox; registrar requisições/respostas anonimizadas e métricas de integração.

## Evidências

- Link para ADR/RFC e documentação oficial usada.
- Testes automatizados para assinatura, idempotência, retries e transições.
- Evidência de sandbox com tokens/IDs sensíveis removidos.