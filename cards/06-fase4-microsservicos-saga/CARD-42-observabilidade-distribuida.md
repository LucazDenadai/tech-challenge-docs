# CARD-42 — Observabilidade do fluxo distribuído

**Tipo:** Observabilidade / Operação
**Status:** To Do
**Depende de:** CARD-36, CARD-40, CARD-41
**Bloqueia:** CARD-43
**Repositórios:** Três serviços, infraestrutura/observabilidade e `tech-challenge-docs`
**Decisão arquitetural:** Reaproveitar ADR-008 e ADR-012; registrar ajustes de instrumentação se necessários

---

## Contexto

O enunciado pede uso das ferramentas de monitoramento e observabilidade implementadas na Fase 3. A instrumentação existente (OpenTelemetry e stack/Datadog conforme ambiente) é reutilizável, mas precisa cobrir os três serviços, pagamentos e Saga. Deploy anterior ou dashboard configurado não prova que traces/alertas do novo fluxo funcionam; a validação deve usar uma demonstração real e dados sintéticos.

## Escopo

- Propagar trace context e correlation/Saga ID em chamadas REST e mensagens assíncronas.
- Padronizar logs estruturados e atributos de serviço/ambiente/filial sem PII ou segredos.
- Métricas de duração/resultado da Saga, pagamentos, execução, fila, retry, DLQ e compensação.
- Dashboards e alertas para falhas e latência do fluxo; links e runbooks de investigação.
- Validar exportação para a stack aprovada da Fase 3 e comportamento quando backend de telemetria está indisponível.

## Critérios de aceite

- [ ] Cada serviço reporta nome/versão/ambiente corretos e traces/logs podem ser filtrados por correlation ID.
- [ ] Uma trace cobre abertura da OS, orçamento, pagamento (incluindo callback), mensagem e execução, com spans de serviço identificáveis.
- [ ] Mensageria preserva trace context/correlation ao atravessar producer/consumer.
- [ ] Existem métricas de sucesso/falha, latência, retries, DLQ e duração/estado das compensações.
- [ ] Dashboard mostra saúde dos três serviços e o estado agregado do fluxo, sem inferir sucesso apenas de disponibilidade HTTP.
- [ ] Alertas/runbooks descrevem ação para pagamento pendente, Saga parada, DLQ crescente e falha de deploy.
- [ ] Segredos, CPF completo, dados de cartão e conteúdo sensível não são exportados em logs/tags.
- [ ] Evidências são capturadas em ambiente autorizado e o custo/janela de ativação está descrito.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Rastrear uma Saga entre microsserviços

  Cenário: Correlacionar traces do fluxo concluído
    Dado que uma OS inicia uma Saga com identificador de correlação
    Quando Billing confirma pagamento e Operações conclui a execução
    Então os spans dos três serviços aparecem no mesmo trace ou contexto correlacionável
    E cada span identifica serviço, operação e resultado

  Cenário: Identificar uma compensação parada
    Dado que uma compensação excedeu o limite de retry definido
    Quando o dashboard de operação é consultado
    Então a Saga aparece em estado de falha/ação necessária
    E há correlation ID, motivo e runbook para investigação

  Cenário: Redigir dados sensíveis
    Dado que uma requisição contém documento pessoal ou segredo de provedor
    Quando logs e traces são emitidos
    Então os valores sensíveis não aparecem nos atributos exportados
```

## Passos

1. Revisar configuração OpenTelemetry/Datadog e estratégia de propagação de contexto.
2. Instrumentar APIs, consumers, outbox, callbacks e chamadas Mercado Pago sem payload sensível.
3. Criar dashboards e alertas focados nos estados do negócio e da Saga.
4. Executar fluxo feliz e cenários de falha; verificar trace, métricas e runbook.
5. Registrar procedimento para ativar/desativar infraestrutura observável de custo variável.

## Evidências

- Link/print do dashboard com dados sintéticos e trace distribuído correlacionado.
- Evidência de alerta para uma falha simulada e runbook correspondente.
- Verificação de ausência de segredos/PII em logs exportados.