# CARD-42 — Observabilidade do fluxo distribuído

**Tipo:** Observabilidade / Operação
**Status:** Concluído — 42a e 42b em 2026-10-13; span do Mercado Pago não verificado ao vivo
**Depende de:** CARD-36, CARD-40, CARD-41
**Bloqueia:** CARD-43
**Repositórios:** Três serviços, infraestrutura/observabilidade e `tech-challenge-docs`
**Decisão arquitetural:** [ADR-022](../../adr/ADR-022-observabilidade-fase4-stack-otel-local.md) (stack OTel da Fase 2/3 no Kind; Datadog fora da Fase 4), ADR-008

---

## Contexto

O enunciado pede uso das ferramentas de monitoramento e observabilidade implementadas na Fase 3. A instrumentação existente (OpenTelemetry e stack/Datadog conforme ambiente) é reutilizável, mas precisa cobrir os três serviços, pagamentos e Saga. Deploy anterior ou dashboard configurado não prova que traces/alertas do novo fluxo funcionam; a validação deve usar uma demonstração real e dados sintéticos.

## Decisão (2026-10-13)

Sem conta AWS (ADR-021) e sem acesso à conta Datadog da Fase 3, o time escolheu a stack OpenTelemetry que a Fase 3 manteve em paralelo ao Datadog: Prometheus, Grafana, Loki com Promtail e Jaeger, no Kind ([ADR-022](../../adr/ADR-022-observabilidade-fase4-stack-otel-local.md)). Onde o escopo abaixo fala em Datadog ou "stack aprovada da Fase 3", vale essa stack.

## Escopo

- Propagar trace context e correlation/Saga ID em chamadas REST e mensagens assíncronas.
- Padronizar logs estruturados e atributos de serviço/ambiente/filial sem PII ou segredos.
- Métricas de duração/resultado da Saga, pagamentos, execução, fila, retry, DLQ e compensação.
- Dashboards e alertas para falhas e latência do fluxo; links e runbooks de investigação.
- Validar exportação para a stack aprovada da Fase 3 e comportamento quando backend de telemetria está indisponível.

## Critérios de aceite

- [x] Cada serviço reporta nome/versão/ambiente corretos e traces/logs podem ser filtrados por correlation ID.
- [x] Uma trace cobre abertura da OS, orçamento, pagamento (incluindo callback), mensagem e execução, com spans de serviço identificáveis. *(Como contexto correlacionável pela tag `correlation_id`: cada ação humana abre um trace; ver [CARD-42b](CARD-42b-plataforma-dashboards-alertas.md).)*
- [x] Mensageria preserva trace context/correlation ao atravessar producer/consumer.
- [x] Existem métricas de sucesso/falha, latência, retries, DLQ e duração/estado das compensações.
- [x] Dashboard mostra saúde dos três serviços e o estado agregado do fluxo, sem inferir sucesso apenas de disponibilidade HTTP.
- [x] Alertas/runbooks descrevem ação para pagamento pendente, Saga parada, DLQ crescente e falha de deploy. *(Disparados no Kind: Saga parada, ação manual, DLQ e serviço fora do ar.)*
- [x] Segredos, CPF completo, dados de cartão e conteúdo sensível não são exportados em logs/tags.
- [x] Evidências são capturadas em ambiente autorizado e o custo/janela de ativação está descrito.

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

## Subcards

- [CARD-42a — Instrumentação dos serviços: identidade, métricas de negócio e logs ligados ao trace](CARD-42a-instrumentacao-servicos.md)
- [CARD-42b — Stack de observabilidade no Kind: dashboards, alertas, runbooks e evidências](CARD-42b-plataforma-dashboards-alertas.md)

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