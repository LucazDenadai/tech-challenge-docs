# CARD-39 — Serviço Billing e pagamentos

**Tipo:** Implementação / Microsserviço
**Status:** To Do
**Depende de:** CARD-34, CARD-35, CARD-36
**Bloqueia:** CARD-40, CARD-41, CARD-43
**Repositório alvo:** Repositório exclusivo de Billing (nome definido no CARD-34)
**Decisão arquitetural:** ADR-014, [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md) e contratos/Saga do CARD-36

---

## Contexto

Billing é owner de orçamento, aprovação e estado financeiro. Deve integrar-se ao Mercado Pago conforme o enunciado, sem permitir que o provedor ou Billing altere diretamente OS ou Operações. A notificação de pagamento precisa ser validada e idempotente. A chave/configuração de sandbox e produção deve ficar protegida e separada.

## Escopo

- Repositório, deploy, infraestrutura e banco independentes.
- Geração de orçamento para OS e registro de aprovação/rejeição.
- Criação e consulta de pagamento junto ao Mercado Pago em sandbox e persistência de estado financeiro.
- Endpoint/webhook de notificação com validação/autenticidade conforme documentação oficial vigente do Mercado Pago.
- Publicação de eventos de orçamento e pagamento conforme CARD-36.
- Ação compensatória de cancelamento/estorno ou fluxo manual de recuperação conforme o estado permitido pelo provedor.

## Fora de escopo

- Gravar diretamente no banco OS ou Operações.
- Armazenar dados de cartão não necessários à integração; não manipular PAN/CVV.
- Declarar pagamento concluído com base apenas no redirect do browser.

## Critérios de aceite

- [ ] Billing persiste orçamentos, pagamentos e reconciliação em sua instância PostgreSQL dedicada e não consulta tabelas de outros serviços.
- [ ] Orçamento pode ser criado, consultado, aprovado ou recusado com histórico e filial/correlation ID.
- [ ] Integração Mercado Pago funciona em sandbox, usando credenciais em secret store/variáveis seguras.
- [ ] Webhook valida autenticidade e consulta/valida estado conforme fluxo oficial adotado, sem confiar cegamente no payload recebido.
- [ ] Notificação duplicada ou fora de ordem não duplica cobrança, evento nem transição financeira.
- [ ] Sucesso, recusa, cancelamento, timeout e erro do provedor são estados distintos e observáveis.
- [ ] Uma cobrança só pode ser criada para a versão vigente de orçamento aprovada e não expirada.
- [ ] A confirmação do provedor só aprova o pagamento quando referência, OS, orçamento, valor e moeda correspondem à tentativa persistida.
- [ ] Recusa de negócio do provedor é distinguida de indisponibilidade/timeout; falha técnica não é registrada como pagamento recusado.
- [ ] Notificação atrasada ou fora de ordem não regride estado terminal nem autoriza a execução depois de uma recusa/cancelamento.
- [ ] Existe regra aprovada para múltiplas tentativas de pagamento; não é possível liquidar mais de uma cobrança para o mesmo orçamento sem decisão explícita.
- [ ] Operação de compensação está implementada ou há procedimento manual auditável para situação não compensável.
- [ ] Testes não dependem de chamadas pagas/live; sandbox e doubles são isolados e o contrato real tem teste de integração controlado.
- [ ] Swagger explica endpoints e exemplos, sem segredos ou dados reais.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Orçar e cobrar uma ordem

  Cenário: Aprovar orçamento e iniciar pagamento
    Dado que Billing recebeu uma solicitação válida de orçamento para uma OS
    Quando o cliente aprova o orçamento
    Então Billing cria uma tentativa de pagamento no Mercado Pago sandbox
    E persiste a referência do provedor em seu próprio banco
    E publica o estado inicial de pagamento com a correlação da OS

  Cenário: Confirmar pagamento por notificação válida
    Dado que Mercado Pago envia uma notificação autenticada
    E o estado consultado no provedor é aprovado
    Quando Billing processa a notificação
    Então registra o pagamento aprovado uma única vez
    E publica o evento de pagamento aprovado

  Cenário: Rejeitar notificação não autenticada
    Dado que Billing recebe uma notificação sem autenticidade válida
    Quando a requisição é processada
    Então nenhum pagamento é marcado como aprovado
    E o evento de pagamento aprovado não é publicado

  Cenário: Não iniciar pagamento para orçamento não aprovado ou expirado
    Dado que o orçamento da OS está pendente, rejeitado ou expirado
    Quando é solicitada a criação de uma cobrança
    Então Billing rejeita a solicitação com o resultado definido no contrato
    E nenhuma cobrança é criada no Mercado Pago
    E nenhum evento de pagamento aprovado ou autorização de execução é publicado

  Cenário: Não confirmar pagamento com valor ou moeda divergentes
    Dado que uma notificação autenticada referencia uma tentativa de pagamento existente
    Mas o valor ou a moeda consultados no Mercado Pago diferem do orçamento aprovado
    Quando Billing reconcilia o estado com o provedor
    Então o pagamento não é marcado como aprovado
    E a divergência é registrada para investigação
    E nenhum evento de autorização de execução é publicado

  Cenário: Diferenciar recusa do provedor de falha técnica
    Dado que o Mercado Pago não concluiu a solicitação de pagamento
    Quando o provedor retorna uma recusa de negócio
    Então Billing registra a tentativa como recusada
    E não publica pagamento aprovado
    Mas quando o provedor está indisponível ou a chamada expira
    Então Billing mantém a tentativa em estado pendente ou de resultado desconhecido
    E aplica retry/reconciliação sem inventar uma recusa

  Cenário: Ignorar notificação atrasada que regrediria o estado
    Dado que um pagamento já foi confirmado como aprovado ou estornado
    Quando chega uma notificação válida de estado anterior
    Então Billing não regride o estado financeiro
    E não publica novamente um evento incompatível com o estado terminal

  Cenário: Rejeitar tentativa de estorno de pagamento não aprovado
    Dado que a tentativa de pagamento foi recusada ou não chegou a ser criada
    Quando a Saga solicita um estorno
    Então Billing não envia um estorno inválido ao provedor
    E responde com o resultado compensatório idempotente definido no contrato
```

## Subcards

- [CARD-39a — Orçamento e aprovação](CARD-39a-orcamento-aprovacao.md)
- [CARD-39b — Integração Mercado Pago e webhook](CARD-39b-mercado-pago-webhook.md)

## Evidências

- Testes, workflow, OpenAPI e coleção Postman.
- Execução em sandbox documentada com identificadores anonimizados e sem chaves.
- Prova de webhook duplicado/inválido e cenário de compensação/recuperação.