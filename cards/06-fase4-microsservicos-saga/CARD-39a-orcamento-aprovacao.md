# CARD-39a — Orçamento e aprovação no Billing

**Tipo:** Implementação / Domínio
**Status:** To Do
**Depende de:** CARD-35, CARD-36
**Bloqueia:** CARD-39b, CARD-40
**Repositório alvo:** Repositório exclusivo de Billing
**Decisão arquitetural:** ADR-014 e catálogo de contratos do CARD-36

---

## Contexto

O enunciado inclui geração e envio de orçamento para aprovação. Billing deve ser responsável pelo ciclo de orçamento e por manter a decisão financeira associada à ordem, enquanto OS mantém apenas o estado geral e Operações só recebe uma autorização de fluxo válida.

## Critérios de aceite

- [ ] Orçamento tem versão, valor/moeda, itens referenciados, validade, filial e estado de aprovação.
- [ ] Criação não lê tabelas de OS/Operações; os dados necessários chegam via contrato aprovado.
- [ ] Aprovação/rejeição autenticada e repetida é idempotente e mantém histórico/auditoria.
- [ ] Aprovação se aplica à versão exata apresentada ao cliente; decisão para versão antiga é rejeitada se o orçamento foi alterado.
- [ ] Somente orçamento vigente e aprovado pode iniciar pagamento; orçamento pendente, rejeitado, cancelado ou expirado não pode gerar cobrança nem liberar execução.
- [ ] Decisão sem autorização válida não altera o estado do orçamento nem publica evento de aprovação/rejeição.
- [ ] Mudança de orçamento após aprovação segue uma regra explícita e reinicia autorização/pagamento quando aplicável.
- [ ] Resultado é publicado via evento conforme CARD-36 e não altera banco OS diretamente.
- [ ] Testes cobrem validade, aprovação, rejeição, expiração e concorrência de decisão.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Aprovar orçamento de uma OS

  Cenário: Cliente aprova orçamento válido
    Dado que existe um orçamento pendente e não expirado
    Quando o cliente autorizado aprova a versão apresentada
    Então Billing registra a aprovação dessa versão
    E publica o evento de aprovação uma única vez

  Cenário: Aprovação de orçamento expirado
    Dado que o orçamento expirou
    Quando o cliente tenta aprová-lo
    Então Billing rejeita a decisão
    E a OS não é liberada para execução

  Cenário: Rejeitar decisão sobre uma versão antiga
    Dado que o cliente recebeu a versão 1 do orçamento
    E Billing publicou uma versão 2 antes da resposta do cliente
    Quando chega uma aprovação referente à versão 1
    Então Billing rejeita a decisão como desatualizada
    E mantém a versão 2 aguardando decisão
    E não publica evento de orçamento aprovado

  Cenário: Rejeitar decisão de usuário sem autorização
    Dado que o orçamento está pendente
    E o solicitante não pode decidir por essa OS
    Quando tenta aprovar ou rejeitar o orçamento
    Então Billing rejeita a solicitação
    E mantém estado e histórico de decisão inalterados

  Cenário: Não criar cobrança para orçamento rejeitado
    Dado que o cliente rejeitou o orçamento
    Quando o fluxo tenta solicitar um pagamento
    Então Billing não cria uma tentativa no provedor
    E publica ou retorna somente o resultado de rejeição previsto no contrato
    E Operações não recebe autorização para iniciar a execução
```

## Passos

1. Modelar orçamento e histórico como agregado de Billing.
2. Definir identidade/ autorização do aprovador sem transportar dados de pagamento sensíveis.
3. Implementar API e eventos, depois testes de integração.
4. Atualizar Swagger e collection com exemplos fictícios.

## Evidências

- Testes unitários/integração, OpenAPI e amostra de evento versionado.