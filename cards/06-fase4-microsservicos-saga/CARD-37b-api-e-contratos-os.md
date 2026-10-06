# CARD-37b — API, contratos e histórico da OS

**Tipo:** Implementação / API
**Status:** To Do
**Depende de:** CARD-36, CARD-37a
**Bloqueia:** CARD-40, CARD-43
**Repositório alvo:** `tech-challenge-os`
**Decisão arquitetural:** Contratos aprovados no CARD-36 ([ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md), [ADR-018](../../adr/ADR-018-contratos-assincronos-asyncapi-fase4.md))

---

## Contexto

OS é a autoridade sobre a ordem e precisa expor operações síncronas necessárias aos clientes e receber eventos dos serviços parceiros. O contrato deve preservar a propriedade do estado e permitir correlação de chamadas/mensagens sem acoplar a API ao esquema de dados de outro serviço.

## Decisões de implementação (2026-10-06)

- **API:** o projeto executável `OficinaMecanica.OS.API` expõe toda a API sob ownership do OS, não só a OS: auth/JWT dos funcionários, clientes, veículos, usuários, abertura/listagem/detalhe da OS, status, histórico, transições manuais (`Entregue`, cancelamento), endpoint interno de busca de cliente por CPF para a Lambda (ADR-015), health/readiness e Swagger. O `Program.cs` chama o seed de demonstração com `Seed:SenhaUsuarios`.
- **Eventos (fronteira com o CARD-40):** o 37b entrega só a infraestrutura de consumo dos canais que o OS recebe no AsyncAPI: conexão autenticada ao RabbitMQ, declaração das filas `os.<canal>` e das DLQs `os.<canal>.dlq`, validação do envelope e do payload, deduplicação por `messageId` em uma tabela inbox no PostgreSQL do OS, envio para a DLQ com motivo e propagação do `correlationId` em logs e traces. O efeito de negócio (transição da Saga e do status da OS), a outbox e a publicação de comandos ficam no CARD-40, porque o ADR-017 não permite escrever o status fora da transição da Saga.

## Critérios de aceite

- [ ] Endpoints cobrem cadastros, auth, abertura, consulta de status e consulta de histórico com validação, autorização e códigos HTTP documentados.
- [ ] Endpoint interno de busca por CPF é autenticado e devolve só o necessário para a Lambda.
- [ ] API tem Swagger/OpenAPI atualizado e exemplos sem credenciais ou dados pessoais reais.
- [ ] Eventos recebidos seguem o schema aprovado, são autenticados conforme infraestrutura e têm correlation ID.
- [ ] Eventos duplicados ou incompatíveis não geram mutação silenciosa: duplicata é confirmada sem novo registro; tipo/versão desconhecido ou payload inválido vai para a DLQ com motivo e log.
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

  Cenário: Reentregar um evento já recebido
    Dado que o OS já registrou um evento com determinado messageId
    Quando a mesma mensagem é entregue novamente
    Então a mensagem é confirmada sem criar novo registro

  Cenário: Receber evento incompatível
    Dado uma mensagem com tipo, versão ou payload fora do contrato
    Quando o OS a consome
    Então a mensagem vai para a DLQ do canal com o motivo
    E nenhum estado do OS é alterado
```

## Passos

1. Criar o projeto API com controllers, DTOs, autorização, validação, health e Swagger.
2. Implementar o consumidor dos canais de Billing e Operações definidos no CARD-36: envelope, inbox, DLQ e correlation ID (sem efeito de negócio, que fica no CARD-40).
3. Adicionar logs estruturados e trace context.
4. Publicar Swagger e casos de exemplo para Postman da solução.

## Evidências

- OpenAPI gerado, testes de API e coleção Postman atualizada.
- Trace de exemplo com correlation ID propagado.
