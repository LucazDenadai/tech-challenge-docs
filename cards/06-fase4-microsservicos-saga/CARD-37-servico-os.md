# CARD-37 — Serviço OS independente

**Tipo:** Implementação / Microsserviço
**Status:** To Do
**Depende de:** CARD-34, CARD-35, CARD-36
**Bloqueia:** CARD-40, CARD-41, CARD-43
**Repositório alvo:** Repositório exclusivo do serviço OS (nome definido no CARD-34)
**Decisão arquitetural:** [ADR-014](../../adr/ADR-014-limites-microsservicos-fase4.md) e ADR de persistência aprovado no CARD-35

---

## Contexto

O atual Atendimento contém capacidades que passarão a pertencer a OS, Billing e Operações. Este card entrega OS como serviço independente, preservando o comportamento necessário de abertura, consulta, histórico e estado geral, sem acesso direto a bancos externos. A extração deve ser incremental, com contratos do CARD-36 e evidência de compatibilidade.

## Escopo

- Criar ou preparar repositório exclusivo, projeto executável, API, banco próprio, Dockerfile e manifests do serviço.
- Migrar apenas entidades, regras e casos de uso cujo owner seja OS.
- Implementar abertura de OS e consulta de status/histórico; consumir os resultados de orçamento/pagamento e progresso de Operações pelos contratos aprovados.
- Manter autorização e associação de filial conforme decisões transversais.
- Fornecer health/readiness, Swagger/OpenAPI e telemetria com correlation ID.
- Remover qualquer necessidade de consultar diretamente banco de Billing ou Operações.

## Fora de escopo

- Implementar lógica de orçamento, pagamento, estoque ou fila de reparos dentro de OS.
- Coordenar a Saga neste serviço antes de o desenho escolhido no CARD-36 ser implementado no CARD-40.

## Critérios de aceite

- [ ] O serviço compila, inicia localmente/containerizado e é implantável independentemente.
- [ ] O banco pertence exclusivamente ao serviço e usa somente a tecnologia aprovada no CARD-35.
- [ ] Abertura, consulta de estado e histórico são cobertos por testes unitários e integração do serviço.
- [ ] OS persiste apenas seu estado; dados externos chegam por contrato e são tratados de forma idempotente.
- [ ] A API publicada corresponde ao OpenAPI e expõe os endpoints exigidos pelo fluxo.
- [ ] Falha/duplicidade na entrega de evento não cria transições ou registros duplicados.
- [ ] A filial da ordem é preservada nos registros e nas mensagens pertinentes.
- [ ] Nenhuma connection string, senha ou token está versionado.

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