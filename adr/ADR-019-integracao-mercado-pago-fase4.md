# ADR-019 — Integração do Billing com o Mercado Pago (Pix pela API de Pagamentos)

**Status:** Aceito
**Data:** 2026-10-13
**Autores:** Time Tech Challenge — Fase 4
**Relaciona-se a:** [ADR-015](ADR-015-ownership-e-infraestrutura-fase4.md), [ADR-016](ADR-016-bancos-sql-nosql-fase4.md), [ADR-017](ADR-017-saga-orquestrada-os-fase4.md), [ADR-018](ADR-018-contratos-assincronos-asyncapi-fase4.md), [CARD-39b](../cards/06-fase4-microsservicos-saga/CARD-39b-mercado-pago-webhook.md)

---

## Contexto

O ADR-017 definiu que Billing cria a tentativa de pagamento no Mercado Pago, recebe o webhook e só publica `PaymentApproved` depois de reconciliar com o provedor. Ele deixou para o CARD-39b a escolha do produto/API, da autenticação do webhook, da idempotência, do estorno e da expiração (risco "API do Mercado Pago tem estados/limites específicos"). O ADR-015 deixou para o mesmo card a minimização dos dados do pagador.

Restrições do projeto:

- Não existe front-end. O cliente aprova o orçamento pela API (CARD-39a) e precisa pagar sem que Billing receba dados de cartão.
- A Saga precisa da referência do provedor já no `PaymentPending` e de uma janela de pagamento de 30 minutos (ADR-017).
- O time ainda não tem credenciais do Mercado Pago. A integração precisa ser testável sem rede e sem conta, e trocar para o sandbox só por configuração.

---

## Decisão

### Produto: Pix pela API de Pagamentos (Checkout Transparente)

- Billing cria a cobrança com `POST /v1/payments`, `payment_method_id = pix`. A resposta já traz o `id` do pagamento no provedor, o código Pix "copia e cola" (`point_of_interaction.transaction_data.qr_code`) e a página de pagamento (`ticket_url`).
- `external_reference` = `paymentId` interno. `date_of_expiration` = criação + `MercadoPago:ExpiracaoMinutos` (padrão 30, o mínimo aceito pelo Pix). Pix não pago até lá passa a `cancelled` no provedor, e Billing trata como recusa.
- O cliente consulta o código Pix pela API de Billing (`GET /billing/pagamentos/...`, mesmo JWT do orçamento). O código não sai em mensagens da Saga.

### Autenticação

- **Chamadas ao provedor:** `Authorization: Bearer <MercadoPago:AccessToken>`. O token vem de variável de ambiente/Secret (CARD-41) e nunca entra em log, trace ou resposta.
- **Webhook:** Billing valida o cabeçalho `x-signature` (`ts=<epoch>,v1=<hmac>`). O manifesto é `id:<data.id>;request-id:<x-request-id>;ts:<ts>;`, com `data.id` da query string em minúsculas. O HMAC-SHA256 usa `MercadoPago:WebhookSecret`, e a comparação é em tempo constante. Assinatura ausente ou inválida responde `401`, gera log de aviso e métrica, e não muda estado.
- **O corpo do webhook nunca é fonte de estado.** Uma notificação válida só dispara a consulta `GET /v1/payments/{id}`. Assim, um replay de notificação válida custa uma consulta, mas não altera nada. O `ts` aceita até `MercadoPago:ToleranciaAssinaturaMinutos` (padrão 30) de diferença. A janela é larga porque o Mercado Pago reenvia a notificação a cada 15 minutos, e a proteção principal é a consulta autoritativa.

### Estados e reconciliação

Estados internos do `Pagamento`: `Solicitado` → `Pendente` → `Aprovado` ou `Recusado`; depois de `Aprovado`: `EstornoPendente` → `Estornado` ou `EstornoFalhou`. As transições só avançam; notificação atrasada ou fora de ordem não regride estado.

| Status no Mercado Pago | Estado em Billing | Evento (uma vez, na transição) |
|---|---|---|
| (criação sem resposta: timeout, 5xx, 429, rede) | `Solicitado` | `PaymentOutcomeUnknown` |
| `pending`, `in_process`, `authorized` | `Pendente` | `PaymentPending` |
| `approved` | `Aprovado` | `PaymentApproved` |
| `rejected`, `cancelled` | `Recusado` | `PaymentDeclined` |
| `refunded` (após estorno pedido pela Saga) | `Estornado` | `PaymentRefunded` |
| `refunded`/`charged_back` sem pedido da Saga, `in_mediation` | sem mudança | nenhum; log de erro e métrica para ação manual |

Antes de qualquer transição, Billing confere o pagamento consultado: `external_reference`, `id` do provedor (se já gravado), valor e moeda iguais aos da tentativa interna. Divergência ou pagamento desconhecido não muda estado, não publica evento e fica registrado para investigação.

Um serviço em segundo plano reconcilia o que não é terminal: refaz a criação de `Solicitado` (mesma chave idempotente), consulta `Pendente` sem notificação recente e confirma `EstornoPendente`. Depois de `MercadoPago:MaxTentativasReconciliacao` (padrão 10) falhas técnicas seguidas, o pagamento é marcado para reconciliação manual, com log de erro e métrica; não há DLQ para chamadas HTTP.

### Idempotência, retry e rate limit

- Cada `PaymentCreationRequested` gera no máximo um `Pagamento`; a `idempotencyKey` do comando é única na tabela. O `X-Idempotency-Key` enviado ao provedor é o `paymentId`, então todo retry da mesma tentativa reutiliza a mesma chave e não cria segunda cobrança.
- O estorno usa `X-Idempotency-Key = estorno-<paymentId>` e é total (ADR-017). Um `PaymentRefundRequested` repetido devolve o resultado já gravado sem nova chamada.
- Timeout de 10 s por chamada. Até 3 tentativas imediatas com espera exponencial para timeout, erro de rede, `5xx` e `429`; depois disso, o resultado fica desconhecido e segue para a reconciliação. Erro HTTP nunca vira recusa.

### Dados do pagador

- O Pix exige `payer.email`. O `PaymentCreationRequested.v1` ganha o campo **opcional** `payerEmail`; pela regra do ADR-018, campo opcional novo é mudança compatível e mantém a versão. O OS envia o e-mail do cadastro do cliente (CARD-40).
- Billing guarda o e-mail só enquanto a cobrança não existe no provedor (para a reconciliação refazer a criação) e o apaga ao chegar em `Pendente`. CPF, nome e dados de cartão não chegam a Billing.
- Comando sem `payerEmail`, para orçamento não aprovado, ou com valor/moeda diferentes da versão aprovada, vai para a DLQ com o motivo e não cria cobrança. Esses casos indicam erro do orquestrador, não recusa de pagamento.

### Provedor por configuração

- O provedor fica atrás da porta `IProvedorPagamento`. `MercadoPago:Modo = Simulado` usa um provedor em memória com idempotência e mudança de status controlada por rota de teste (só nesse modo). `Modo = MercadoPago` usa o cliente HTTP real (sandbox ou produção pelas credenciais).
- O cliente HTTP é escrito com `HttpClient`, sem o SDK oficial, para controlar cabeçalhos, timeout, retry e testes com respostas simuladas.

---

## Alternativas consideradas

### Checkout Pro (preferência com redirecionamento)

**Não escolhida:** também evita dados de cartão em Billing, mas o pagamento só existe quando o cliente abre o link. O `PaymentPending` teria só o ID da preferência, e a reconciliação teria de buscar pagamentos por preferência ou por `merchant_order`. Continua válida se o projeto ganhar um front-end com redirecionamento.

### Cartão pela API de Pagamentos

**Rejeitada no MVP:** exige tokenizar o cartão no navegador (Card Payment Brick/SDK JS). Sem front-end, não há onde coletar o cartão sem que Billing toque nos dados.

### API de Orders do Mercado Pago

**Não escolhida agora:** a documentação do Mercado Pago apresenta a API de Orders como a evolução das integrações. A API de Pagamentos continua documentada para Pix, com idempotência, estorno e expiração, que é o que a Saga precisa. A troca fica isolada no adapter atrás de `IProvedorPagamento`.

### Confiar no corpo do webhook

**Rejeitada:** o corpo só traz o ID do recurso e pode ser reenviado ou forjado. O ADR-017 exige reconciliação com o provedor antes de publicar aprovação.

---

## Consequências

### Positivas

- Nenhum dado de cartão passa por Billing; o escopo PCI fica no Mercado Pago.
- A Saga recebe a referência do provedor já no `PaymentPending`, e a expiração do Pix coincide com a janela de 30 minutos do ADR-017.
- O fluxo inteiro, incluindo aprovação, recusa e estorno, roda localmente e no vídeo sem credenciais, pelo provedor simulado.

### Negativas e riscos

| Risco | Mitigação |
|---|---|
| Sandbox do Pix não permite pagar de verdade | Aprovação demonstrada no modo `Simulado`; no sandbox, a evidência cobre criação, consulta, assinatura do webhook e estorno com o que o ambiente permitir (CARD-39b, passo 6). |
| Webhook precisa de URL pública | Até o deploy (CARD-41), validar com túnel local ou só com o provedor simulado; a reconciliação por consulta funciona mesmo sem webhook. |
| E-mail do cliente trafega na mensagem | Só o e-mail, nada de CPF; apagado de Billing ao criar a cobrança; mensagens não entram em log com payload. |
| API de Pagamentos ser descontinuada em favor de Orders | Adapter isolado e testes de contrato do adapter com respostas gravadas. |
| Estorno feito direto no painel do Mercado Pago | Não muda estado sozinho; reconciliação registra a divergência para ação manual. |

---

## Referências

- [Integrar Pix — API de Pagamentos](https://www.mercadopago.com.br/developers/pt/docs/checkout-api-payments/integration-configuration/integrate-pix)
- [Webhooks e validação de assinatura](https://www.mercadopago.com.br/developers/en/docs/your-integrations/notifications/webhooks)
- [Reembolso de transações com Pix](https://www.mercadopago.com.br/developers/pt/docs/checkout-api-payments/payment-management/cancellations-and-refunds/refund-pix)
- [ADR-017 — Saga orquestrada pelo serviço OS](ADR-017-saga-orquestrada-os-fase4.md)
- [CARD-39b — Mercado Pago: cobrança e webhook](../cards/06-fase4-microsservicos-saga/CARD-39b-mercado-pago-webhook.md)
