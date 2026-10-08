# CARD-37 — Serviço OS independente

**Tipo:** Implementação / Microsserviço
**Status:** Em andamento — CARD-37a e CARD-37b concluídos; falta imagem/deploy independente e CI (CARD-41) e a filial nas mensagens publicadas (CARD-40)
**Depende de:** CARD-34, CARD-35, CARD-36
**Bloqueia:** CARD-40, CARD-41, CARD-43
**Repositório alvo:** `tech-challenge-os`
**Decisão arquitetural:** [ADR-014](../../adr/ADR-014-limites-microsservicos-fase4.md), [ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md), [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md), [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md), [ADR-018](../../adr/ADR-018-contratos-assincronos-asyncapi-fase4.md)

---

## Contexto

O atual Atendimento contém capacidades que passarão a pertencer a OS, Billing e Operações. Este card entrega OS como serviço independente, preservando o comportamento necessário de abertura, consulta, histórico e estado geral, sem acesso direto a bancos externos. A extração deve ser incremental, com contratos do CARD-36 e evidência de compatibilidade.

## Escopo

- Criar ou preparar repositório exclusivo, projeto executável, API, banco próprio, Dockerfile e manifests do serviço.
- Extrair do Atendimento apenas o código (entidades, regras e casos de uso) cujo owner seja OS. Não há migração de dados: banco vazio, migrations próprias e seed de demonstração (emenda do ADR-015).
- Manter os cadastros sob ownership do OS: clientes, veículos, filiais e usuários funcionários (Admin, Atendente, Mecânico), incluindo a emissão do JWT dos funcionários.
- Expor endpoint interno autenticado para a Lambda localizar o cliente por CPF, substituindo a consulta direta da Lambda ao banco (ADR-015).
- Manter a notificação por e-mail ao cliente quando o status da OS muda.
- Implementar abertura de OS e consulta de status/histórico; consumir os resultados de orçamento/pagamento e diagnóstico, reserva, início, conclusão e falha de Operações pelos contratos do AsyncAPI (ADR-018).
- Manter autorização e associação de filial conforme decisões transversais.
- Fornecer health/readiness, Swagger/OpenAPI e telemetria com correlation ID.
- Remover qualquer necessidade de consultar diretamente banco de Billing ou Operações.

## Fora de escopo

- Implementar lógica de orçamento, pagamento, estoque ou fila de reparos dentro de OS.
- Coordenar a Saga neste serviço antes de o desenho escolhido no CARD-36 ser implementado no CARD-40.

## Critérios de aceite

- [ ] O serviço compila, inicia localmente/containerizado e é implantável independentemente. *(Compila e sobe em container do SDK; Dockerfile, manifests e workflow ficam no CARD-41.)*
- [x] O banco pertence exclusivamente ao serviço e usa somente a tecnologia aprovada no CARD-35.
- [x] Abertura, consulta de estado e histórico são cobertos por testes unitários e integração do serviço.
- [x] OS persiste apenas seu estado; dados externos chegam por contrato e são tratados de forma idempotente.
- [x] A API publicada corresponde ao OpenAPI e expõe os endpoints exigidos pelo fluxo.
- [x] Falha/duplicidade na entrega de evento não cria transições ou registros duplicados.
- [ ] A filial da ordem é preservada nos registros e nas mensagens pertinentes. *(OS e inbox gravam `FilialId`; as mensagens publicadas pelo OS ficam no CARD-40.)*
- [x] Nenhuma connection string, senha ou token está versionado.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Gerenciar o ciclo de vida da OS

  Cenário: Abrir uma OS vinculada a filial
    Dado que cliente, veículo e filial são válidos
    Quando a solicitação de abertura é aceita
    Então OS cria a ordem em seu próprio banco
    E retorna um identificador consultável
    E registra a filial associada à ordem

  Cenário: Atualizar a OS com confirmação de pagamento
    Dado que OS recebeu um evento válido e correlacionado de pagamento aprovado
    Quando o evento é processado
    Então OS persiste a transição permitida em seu próprio banco
    E não consulta o banco de Billing

  Cenário: Reentregar confirmação de pagamento
    Dado que a confirmação de pagamento já foi processada
    Quando a mesma mensagem é entregue novamente
    Então a ordem não recebe uma segunda transição nem efeito duplicado
```

## Subcards

- [CARD-37a — Extrair domínio e persistência de OS](CARD-37a-dominio-e-persistencia-os.md)
- [CARD-37b — API, contratos e histórico da OS](CARD-37b-api-e-contratos-os.md)

## Dependências e evidências

Subcards devem estar concluídos antes de fechar este card. Anexar link do repositório, workflow, testes, OpenAPI e execução dos cenários.

Registradas até 2026-10-08:

- Repositório: [tech-challenge-os](https://github.com/LucazDenadai/tech-challenge-os), PRs #1 (37a) e #2 (37b).
- Testes e OpenAPI: tabela de evidências do [CARD-37b](CARD-37b-api-e-contratos-os.md#evidências).
- Execução dos cenários: [collection Postman](../../postman/README.md) e [trace de exemplo](../../evidencias/fase4/README.md).
- Workflow: pendente (CARD-41).