# ADR-002 — Observabilidade de falhas de mensageria via tabela no banco de dados

**Status:** Aceito  
**Data:** 2026-05-26  
**Autores:** Time Tech Challenge — Fase 2

---

## Contexto

Com a implementação do CARD-09 (RabbitMQ + MassTransit), o microserviço Estoque passa a consumir eventos `OsFinalizadaEvent` de forma assíncrona. O MassTransit provê retry automático (3 tentativas com intervalos crescentes de 1s, 5s e 10s) e move mensagens que esgotam as tentativas para a queue `estoque.baixa_error`.

O problema identificado: **não há como responder às perguntas operacionais básicas**:

- "A OS `abc-123` teve sua baixa de estoque processada?"
- "Por que a OS `xyz-456` falhou?"
- "Quantas OS estão com falha de baixa agora?"

A Management UI do RabbitMQ mostra contagem de mensagens na fila de erro, mas não vincula à OS específica nem persiste histórico após o reprocessamento. Os logs do container se perdem ao reiniciar. Não há endpoint consultável por `osId`.

A questão de risco levantada antes de decidir: **gravar falhas no banco pode comprometer o desempenho ou criar um vetor de ataque (flooding)?**

Análise:
- A tabela só recebe `INSERT` após esgotar **todos os retries** — não a cada tentativa. Um evento que falha 3 vezes gera 1 linha, não 3.
- Para travar o banco via falhas em cascata seria necessário um volume de OS com erro contínuo que, nesse cenário, indica problema estrutural muito maior (consumer fora, banco inacessível).
- Se o banco estiver indisponível quando o fault consumer tenta gravar, a mensagem permanece na `_error` queue do RabbitMQ — são camadas complementares, sem perda de dados.
- O campo `OcorridoEm` indexado permite política de retenção futura sem scan completo da tabela.

---

## Decisão

Implementar uma tabela `FalhaProcessamento` no schema `estoque` do PostgreSQL, alimentada por um **fault consumer** do MassTransit (`IConsumer<Fault<OsFinalizadaEvent>>`). O MassTransit invoca esse consumer automaticamente quando todos os retries se esgotam.

Expor os registros via endpoint `GET /estoque/falhas` (lista) e `GET /estoque/falhas/{osId}` (por OS específica).

### Estrutura da tabela

```sql
CREATE TABLE estoque."FalhasProcessamento" (
    "Id"              uuid PRIMARY KEY,
    "EventId"         uuid NOT NULL,
    "OrdemServicoId"  uuid NOT NULL,
    "Erro"            varchar(2000) NOT NULL,
    "PayloadJson"     text NOT NULL,
    "OcorridoEm"     timestamptz NOT NULL
);

CREATE INDEX "IX_FalhasProcessamento_OcorridoEm"
    ON estoque."FalhasProcessamento" ("OcorridoEm");

CREATE INDEX "IX_FalhasProcessamento_OrdemServicoId"
    ON estoque."FalhasProcessamento" ("OrdemServicoId");
```

### Fluxo de falha

```
BaixaEstoqueConsumer.Consume() lança exceção
        │
        ▼ retry 1 (1s) → ainda falha
        ▼ retry 2 (5s) → ainda falha
        ▼ retry 3 (10s) → ainda falha
        │
        ▼ MassTransit publica Fault<OsFinalizadaEvent>
        │
        ▼ BaixaEstoqueConsumerFaultConsumer.Consume()
           → grava FalhaProcessamento no banco
           → mensagem removida da _error queue
```

---

## Prós

| # | Benefício |
|---|---|
| 1 | **Consultável por negócio** — é possível responder "a OS X teve baixa?" via API, sem acesso ao RabbitMQ ou logs |
| 2 | **Persistente** — sobrevive a restart de containers, diferente dos logs de console |
| 3 | **Sem dependência nova** — reutiliza PostgreSQL já existente e EF Core já configurado |
| 4 | **Integra com o domínio** — a falha fica no mesmo banco que as peças e movimentações, facilitando queries de correlação |
| 5 | **Pressão mínima** — apenas 1 INSERT por mensagem após esgotar retries, não por tentativa |
| 6 | **Complementar ao RabbitMQ** — se o banco falhar ao gravar, a mensagem permanece na `_error` queue; não há perda |

---

## Contras

| # | Risco | Mitigação adotada |
|---|---|---|
| 1 | **Crescimento ilimitado da tabela** sem política de retenção | Campo `OcorridoEm` indexado — purge futuro via `DELETE WHERE OcorridoEm < NOW() - INTERVAL '90 days'` sem scan completo |
| 2 | **Falha ao gravar a falha** se banco estiver indisponível | Mensagem permanece na `_error` queue do RabbitMQ; operador tem dois lugares para verificar |
| 3 | **Não mostra tentativas intermediárias** — só o estado final após todos os retries | Aceitável para o contexto: o que importa operacionalmente é saber quais OS precisam de intervenção |
| 4 | **Reprocessamento manual necessário** — gravar a falha não resolve o problema | Endpoint de reprocessamento (`POST /estoque/falhas/{id}/reprocessar`) pode ser adicionado em iteração futura |
| 5 | **Observabilidade parcial** — informa o erro mas não o caminho completo da requisição | Limitação inerente à abordagem; rastreamento distribuído completo exige solução dedicada (ver seção de escala) |

---

## Alternativas consideradas

### Alternativa 1 — Apenas logs estruturados (Serilog + arquivo)

Substituir o logger padrão por Serilog com sink em arquivo JSON persistido via volume Docker.

**Prós:** sem pressão no banco, logs pesquisáveis por `osId` via grep/jq.

**Contras:** não é consultável via API; requer acesso ao filesystem ou stack de log (Loki, Elastic) para busca eficiente; logs em arquivo crescem igualmente sem rotação configurada.

**Por que não escolhemos:** não resolve a pergunta "a OS X foi processada?" de forma operacional — exige acesso técnico ao servidor e conhecimento de ferramentas de busca em log.

---

### Alternativa 2 — OpenTelemetry + Jaeger

Instrumentar os serviços com OpenTelemetry e exportar traces para Jaeger ou Zipkin.

**Prós:** rastreamento distribuído completo, correlação entre Atendimento e Estoque, UI rica para investigação.

**Contras:** adiciona 2-3 containers ao ambiente (collector, backend, UI); curva de aprendizado significativa; over-engineering para o estágio atual do projeto.

**Por que não escolhemos:** o benefício não justifica a complexidade adicional neste momento. É a evolução natural quando o volume e a criticidade aumentarem.

---

### Alternativa 3 — Dead Letter Queue monitorada via script externo

Manter mensagens na `_error` queue e criar um job periódico que lê e exporta para um arquivo ou dashboard.

**Prós:** sem impacto no banco.

**Contras:** infraestrutura adicional (job agendado, storage), não consultável via API do serviço, acoplamento à estrutura interna do RabbitMQ.

**Por que não escolhemos:** adiciona peça de infraestrutura sem vantagem real sobre a tabela para o contexto atual.

---

## Recomendação para escala

Quando o volume de eventos justificar (estimativa: acima de 10.000 eventos/dia ou múltiplos serviços consumidores), substituir ou complementar esta solução com:

**OpenTelemetry + Grafana + Tempo**

```
Serviços instrumentados com OTel SDK
        │
        ▼ OpenTelemetry Collector
        │
        ├── Traces → Grafana Tempo (rastreamento distribuído)
        ├── Métricas → Prometheus → Grafana (dashboards)
        └── Logs → Loki → Grafana (busca de logs)
```

Benefícios na escala:
- Correlação automática entre eventos do Atendimento e do Estoque via `TraceId`
- Alertas configuráveis (ex: mais de 10 falhas em 5 minutos → notificação)
- Retenção configurável por tier (hot/cold storage)
- Sem pressão no banco transacional — dados de observabilidade ficam em storage separado

A tabela `FalhaProcessamento` pode coexistir como camada de domínio (visível ao negócio via API) enquanto o OTel cuida da observabilidade técnica.

---

## Consequências

- O microserviço Estoque ganha uma migration nova (`AddFalhaProcessamento`)
- O `Program.cs` do Estoque registra `BaixaEstoqueConsumerFaultConsumer` no MassTransit
- Um novo endpoint `GET /estoque/falhas` e `GET /estoque/falhas/{osId}` passa a existir
- A tabela não recebe dados de sucesso — permanece vazia em operação normal, o que é o comportamento esperado
- Operadores podem monitorar a saúde do processamento verificando se a tabela acumula registros

---

## Referências

- [MassTransit — Fault Consumers](https://masstransit.io/documentation/concepts/exceptions#fault-consumers)
- [MassTransit — Error Queues](https://masstransit.io/documentation/concepts/exceptions#error-pipe)
- [OpenTelemetry .NET](https://opentelemetry.io/docs/instrumentation/net/)
- [Grafana Tempo — distributed tracing](https://grafana.com/oss/tempo/)
