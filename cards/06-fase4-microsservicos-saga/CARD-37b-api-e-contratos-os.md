# CARD-37b — API, contratos e histórico da OS

**Tipo:** Implementação / API
**Status:** To Do
**Depende de:** CARD-36, CARD-37a
**Bloqueia:** CARD-40, CARD-43
**Repositório alvo:** Repositório exclusivo de OS
**Decisão arquitetural:** Contratos aprovados no CARD-36

---

## Contexto

OS é a autoridade sobre a ordem e precisa expor operações síncronas necessárias aos clientes e receber eventos dos serviços parceiros. O contrato deve preservar a propriedade do estado e permitir correlação de chamadas/mensagens sem acoplar a API ao esquema de dados de outro serviço.

## Critérios de aceite

- [ ] Endpoints cobrem abertura, consulta de status e consulta de histórico com validação, autorização e códigos HTTP documentados.
- [ ] API tem Swagger/OpenAPI atualizado e exemplos sem credenciais ou dados pessoais reais.
- [ ] Eventos recebidos seguem o schema aprovado, são autenticados conforme infraestrutura e têm correlation ID.
- [ ] Eventos duplicados, atrasados ou incompatíveis não geram mutação silenciosa; rejeições e reprocessamento são observáveis.
- [ ] Mudanças de contrato seguem estratégia de versionamento/compatibilidade aprovada.
- [ ] Testes de API cobrem happy path, validação, autorização e cenários de falha.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Consultar a OS sem expor bancos internos

  Cenário: Consultar status e histórico
    Dado que existe uma OS no banco do serviço OS
    Quando um consumidor autorizado consulta status e histórico pela API
    Então recebe a situação atual e as transições permitidas para exibição
    E a resposta não expõe identificadores internos de banco ou credenciais

  Cenário: Solicitação inválida de abertura
    Dado que a requisição não contém os campos obrigatórios do contrato
    Quando o cliente solicita a abertura da OS
    Então a API retorna erro de validação documentado
    E nenhuma OS é persistida
```

## Passos

1. Definir DTOs e endpoints conforme OpenAPI/contratos aprovados.
2. Implementar handlers de eventos de Billing e Operações definidos no CARD-36.
3. Adicionar autorização, validação, logs estruturados e trace context.
4. Publicar Swagger e casos de exemplo para Postman da solução.

## Evidências

- OpenAPI gerado, testes de API e coleção Postman atualizada.
- Trace de exemplo com correlation ID propagado.