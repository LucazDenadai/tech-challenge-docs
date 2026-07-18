# ADR-001 — Arquitetura de Microserviços com Mensageria Assíncrona

**Status:** Aceito  
**Data:** 2026-05-18  
**Autores:** Time Tech Challenge — Fase 2

---

## Contexto

Na Fase 1 entregamos um monolito em ASP.NET Core com arquitetura em camadas (Domain / Application / Infrastructure / API), banco PostgreSQL e containerização via Docker. O sistema gerencia ordens de serviço (OS), clientes, veículos, peças e serviços de uma oficina mecânica.

Com o crescimento para múltiplas unidades e aumento de volume de OS, identificamos três problemas concretos que a arquitetura atual não resolve bem:

1. **Escala granular impossível** — em horários de pico, o volume de novas OS e consultas de status é muito maior do que o volume de movimentações de estoque. Hoje subir mais instâncias da API significa subir tudo junto, incluindo partes que não precisam escalar.

2. **Acoplamento de domínios distintos** — Estoque (peças, quantidades) e Atendimento (OS, clientes) têm ciclos de vida, regras de negócio e ritmos de mudança completamente diferentes. Tê-los no mesmo processo força deploys conjuntos e aumenta o risco de regressão.

3. **Operação de estoque é eventual por natureza** — quando uma OS é finalizada, a baixa no estoque não precisa acontecer na mesma transação HTTP. O que importa é que ela *aconteça*, com garantia de entrega. Uma falha temporária no serviço de estoque não deve impedir a finalização de uma OS.

---

## Decisão

Adotar **dois microserviços** com **comunicação assíncrona via RabbitMQ** para o fluxo de baixa de estoque:

### Serviços

| Serviço | Responsabilidade | Banco |
|---|---|---|
| **oficina-atendimento** | OS, clientes, veículos, catálogo de serviços, autenticação | Schema `atendimento` |
| **oficina-estoque** | Peças, quantidades, movimentações de entrada/saída | Schema `estoque` |

Ambos compartilham a mesma instância PostgreSQL no ambiente de desenvolvimento e K8s, separados por schema. Isso reduz custo operacional sem violar o isolamento de dados entre domínios — cada serviço acessa apenas seu próprio schema.

### Fluxo de comunicação

```
┌─────────────────────────────────┐
│       oficina-atendimento       │
│                                 │
│  POST /ordens-servico           │
│  PUT  /ordens-servico/{id}/     │
│       status  (→ Finalizada)    │
│                │                │
│                ▼                │
│  Publica evento:                │
│  "os.finalizada"                │
│  { osId, itens: [{pecaId,qty}] }│
└───────────────┬─────────────────┘
                │
         ┌──────▼──────┐
         │  RabbitMQ   │
         │  Exchange:  │
         │  oficina    │
         │  Queue:     │
         │  estoque.   │
         │  baixa      │
         └──────┬──────┘
                │
┌───────────────▼─────────────────┐
│       oficina-estoque           │
│                                 │
│  Consumer: BaixaEstoqueConsumer │
│  - Valida disponibilidade       │
│  - Subtrai quantidade           │
│  - Registra movimentação        │
│  - Ack/Nack com retry           │
└─────────────────────────────────┘
```

**Atendimento não chama Estoque diretamente via HTTP.** Ele apenas publica o evento e segue. O Estoque processa de forma independente, com retry automático em caso de falha.

### Fluxo de consulta de disponibilidade (síncrono, só na criação da OS)

Na **abertura** da OS, Atendimento faz uma chamada HTTP GET ao Estoque para verificar se as peças solicitadas têm quantidade disponível antes de confirmar a OS. Essa é a única chamada síncrona entre os dois serviços. Se o Estoque estiver indisponível nesse momento, a abertura da OS retorna erro — comportamento aceitável, pois é uma operação de escrita com validação de negócio.

---

## Alternativas consideradas

### Alternativa 1: Monolito modular

Manter um único processo com módulos internos bem delimitados (Atendimento, Estoque, Catálogo), sem comunicação entre processos.

**Prós:**
- Operação simples, deploy único, transações ACID triviais
- Sem latência de rede entre módulos
- Troubleshooting linear

**Contras:**
- Não demonstra escalabilidade granular (requisito do desafio)
- Deploy conjunto — uma mudança no estoque força redeploy do atendimento
- Não prepara o time para separação futura se o negócio crescer

**Por que não escolhemos:** não cumpre o objetivo de demonstrar microserviços e escala independente, mesmo sendo a solução mais simples tecnicamente.

---

### Alternativa 2: Três microserviços (Atendimento + Estoque + Catálogo)

Separar também o catálogo de serviços (tabela de preços) como terceiro serviço.

**Prós:**
- Isolamento máximo de domínios
- Catálogo pode ser cacheado independentemente

**Contras:**
- Catálogo muda raramente e tem leitura simples — não justifica a sobrecarga operacional de um terceiro serviço
- Aumenta o número de chamadas HTTP síncronas na criação de OS (Atendimento precisaria consultar Catálogo além de Estoque)
- Mais manifestos K8s, mais configuração Terraform, mais superfície de falha no CI/CD

**Por que não escolhemos:** custo operacional não justificado pelo benefício. O catálogo permanece dentro de Atendimento como módulo interno.

---

### Alternativa 3: Dois microserviços com comunicação 100% síncrona (HTTP)

Manter dois serviços mas substituir a fila por chamadas REST diretas em todos os fluxos.

**Prós:**
- Mais simples de implementar
- Comportamento determinístico e fácil de debugar

**Contras:**
- Acoplamento temporal: se Estoque cair, a finalização de OS também falha
- Sem retry automático — falhas transitórias viram erros para o usuário
- Não demonstra resiliência, que é um dos objetivos da Fase 2

**Por que não escolhemos:** a finalização de OS é o fluxo mais crítico do sistema. Torná-la dependente da disponibilidade de outro serviço em tempo real aumenta o risco operacional sem necessidade.

---

### Alternativa 4: AWS SQS em vez de RabbitMQ

Usar o SQS que o time já conhece do ambiente de trabalho.

**Prós:**
- Conhecimento prévio do time
- Gerenciado, sem operação de broker

**Contras:**
- Requer credenciais AWS no cluster K8s
- Não roda offline — impossível demonstrar localmente sem mock ou LocalStack
- Adiciona dependência de cloud externa ao ambiente que precisa ser autossuficiente para avaliação
- LocalStack adiciona mais um container no docker-compose

**Por que não escolhemos:** o ambiente de demonstração precisa rodar completo localmente (docker-compose) e no K8s sem dependências externas. RabbitMQ cumpre os mesmos requisitos de mensageria e roda como container.

---

## Consequências

### Positivas
- **Escala independente:** o HPA do K8s pode escalar Atendimento em pico de OS sem tocar o Estoque
- **Resiliência no fluxo crítico:** finalização de OS não falha por indisponibilidade do Estoque
- **Isolamento de deploy:** mudanças no cálculo de estoque não exigem redeploy do Atendimento
- **Retry garantido:** RabbitMQ recoloca a mensagem na fila em caso de falha do consumer, sem perda de evento
- **Demonstração real de microserviços:** dois serviços independentes com contrato explícito via mensagem

### Negativas e mitigações

| Risco | Mitigação |
|---|---|
| Inconsistência eventual: OS finalizada mas estoque não baixado ainda | Aceito por design — a movimentação é registrada com timestamp do evento, auditável |
| Debugging distribuído mais difícil | Correlation ID nos eventos e logs estruturados em ambos os serviços |
| Complexidade na abertura de OS (chamada síncrona ao Estoque) | Circuit breaker simples com Polly; se Estoque indisponível, retorna 503 com mensagem clara |
| Mais manifests K8s para manter | Estrutura padronizada em `/k8s/atendimento/` e `/k8s/estoque/` com templates similares |

---

## Estrutura de repositório resultante

```
Tech-challenge/
├── src/
│   ├── Atendimento/
│   │   ├── OficinaMecanica.Atendimento.Domain/
│   │   ├── OficinaMecanica.Atendimento.Application/
│   │   ├── OficinaMecanica.Atendimento.Infrastructure/
│   │   └── OficinaMecanica.Atendimento.API/
│   └── Estoque/
│       ├── OficinaMecanica.Estoque.Domain/
│       ├── OficinaMecanica.Estoque.Application/
│       ├── OficinaMecanica.Estoque.Infrastructure/
│       └── OficinaMecanica.Estoque.API/
├── tests/
│   ├── Atendimento/
│   │   ├── OficinaMecanica.Atendimento.UnitTests/
│   │   └── OficinaMecanica.Atendimento.IntegrationTests/
│   └── Estoque/
│       ├── OficinaMecanica.Estoque.UnitTests/
│       └── OficinaMecanica.Estoque.IntegrationTests/
├── k8s/
│   ├── atendimento/    # Deployment, Service, HPA, ConfigMap, Secret
│   ├── estoque/        # Deployment, Service, HPA, ConfigMap, Secret
│   └── rabbitmq/       # StatefulSet, Service, PersistentVolumeClaim
├── infra/              # Terraform
├── docs/
│   └── adr/
│       └── ADR-001-arquitetura-microservicos-mensageria.md
└── docker-compose.yml  # postgres + rabbitmq + atendimento + estoque
```

---

## Contrato do evento

```json
// Exchange: oficina.events
// Routing key: os.finalizada
// Queue: estoque.baixa

{
  "eventId": "uuid-v4",
  "eventType": "os.finalizada",
  "occurredAt": "2026-05-18T14:30:00Z",
  "correlationId": "uuid-da-os",
  "payload": {
    "ordemServicoId": "uuid-da-os",
    "numero": "OS-2026-00042",
    "itens": [
      {
        "pecaId": "uuid-da-peca",
        "quantidade": 2
      }
    ]
  }
}
```

---

## Referências

- [Microservices Patterns — Chris Richardson](https://microservices.io/patterns/)
- [RabbitMQ .NET Client documentation](https://www.rabbitmq.com/dotnet.html)
- [MassTransit — abstração de mensageria para .NET](https://masstransit.io/)
- [Polly — resiliência HTTP para .NET](https://www.thepollyproject.org/)
