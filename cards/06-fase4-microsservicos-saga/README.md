# Épico 06 — Fase 4: Microsserviços, Saga e operação nacional

**Status:** Planejamento
**Enunciado:** Tech Challenge — Fase 4 (texto fornecido pelo time em 2026-10-05)
**Decisão de fronteiras:** [ADR-014](../../adr/ADR-014-limites-microsservicos-fase4.md)
**Ownership e repositórios:** [ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md)
**Saga:** [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md) (orquestração pelo OS, aceita)

---

## Objetivo

Evoluir a aplicação da oficina para três microsserviços de negócio independentes, com dados e deploy sob responsabilidade de cada serviço, e demonstrar o fluxo distribuído de abertura, orçamento/aprovação, pagamento, execução e conclusão da OS. A entrega deve incluir Saga com compensações, Mercado Pago, persistência SQL e NoSQL no escopo definido pelo time, testes e evidências exigidas no enunciado.

O começo é deliberadamente simples: fechar decisões e contratos, depois entregar um fluxo vertical mínimo que possa ser demonstrado. Otimizações ou serviços adicionais só entram se houver requisito, risco ou evidência que os justifique.

## Arquitetura-alvo aprovada

| Serviço | Responsabilidades | Repositório de código |
|---|---|---|
| OS | Abertura, estado geral, consulta e histórico da ordem | Nome a definir no CARD-34 |
| Billing | Orçamento, aprovação e pagamento Mercado Pago | Nome a definir no CARD-34 |
| Operações | Estoque **e** execução/produção, incluindo fila, diagnóstico, reparos, saldo e movimentações | Nome a definir no CARD-34 |

Os três serviços têm repositórios de código separados, definição de infraestrutura e banco próprios. Nomes finais, tecnologia/topologia dos bancos e responsabilidade por recursos compartilhados devem ser registrados antes da implementação. Nenhum serviço lê ou grava diretamente no banco de outro.

## Regras de documentação

- Cards usam o formato existente: tipo, status, dependências, decisão arquitetural, contexto, escopo, critérios de aceite, cenários Gherkin, passos e evidências.
- Cenários Gherkin descrevem comportamento e resultado verificável; não substituem testes executáveis. Os cenários do fluxo crítico devem ser automatizados no card de BDD.
- Decisões transversais recebem ADR/RFC neste repositório. Uma alternativa não aprovada fica explicitamente marcada como proposta, não como decisão tomada.
- Cada card nomeia os repositórios de execução envolvidos e os artefatos que devem ser vinculados na evidência.
- O status só muda para concluído depois da verificação documentada, sem inferir conclusão a partir de código ou infraestrutura preexistente.

## Sequência

```text
CARD-33 -> CARD-34 -> CARD-35 -> CARD-36
                         |            |
                         +----------> CARD-37 (OS)
                         +----------> CARD-38 (Operações: Estoque + Execução)
                         +----------> CARD-39 (Billing + Mercado Pago)
                                      |
CARD-37 + CARD-38 + CARD-39 ----------+--> CARD-40 (Saga + BDD integrado)
CARD-34 + CARD-37 + CARD-38 + CARD-39 ----> CARD-41 (CI/CD + Kubernetes)
CARD-40 + CARD-41 --------------------------> CARD-42 (Observabilidade distribuída)
CARD-33 + CARD-40 + CARD-41 + CARD-42 ------> CARD-43 (Documentação e entrega)
```

Subcards mantêm entregas limitadas dentro dos cards de serviço:

- CARD-37a/37b: extração do domínio OS e contrato/API do serviço.
- CARD-38a/38b: capacidade de Estoque e fluxo de Execução no serviço Operações.
- CARD-39a/39b: orçamento/aprovação e integração de pagamentos.

## Cards

Baseline atual dos requisitos, evidências e ambiguidades: [matriz-requisitos.md](matriz-requisitos.md).

| Card | Escopo | Estado |
|---|---|---|
| [CARD-33](CARD-33-mapa-requisitos-e-baseline.md) | Matriz de requisitos, baseline e evidências existentes | Concluído — baseline estático |
| [CARD-34](CARD-34-limites-e-propriedade-dados.md) | Limites, repositórios, infraestrutura e propriedade dos dados | Concluído — decisões registradas |
| [CARD-35](CARD-35-decisao-sql-nosql.md) | Decisão de alocação SQL/NoSQL e topologia dos bancos | Concluído — decisão registrada; implementação pendente |
| [CARD-36](CARD-36-contratos-e-desenho-saga.md) | Contratos, eventos, Saga e compensações | Concluído — estratégia confirmada; implementação pendente |
| [CARD-37](CARD-37-servico-os.md) | Serviço OS | Em andamento — 37a e 37b concluídos; deploy/CI (CARD-41) e publicação de mensagens (CARD-40) pendentes |
| [CARD-38](CARD-38-servico-operacoes.md) | Operações: Estoque + Execução | Em andamento — 38a concluído; 38b a fazer |
| [CARD-39](CARD-39-servico-billing-pagamentos.md) | Billing e Mercado Pago | To Do |
| [CARD-40](CARD-40-saga-e-bdd-integrado.md) | Implementação da Saga e BDD ponta a ponta | To Do |
| [CARD-41](CARD-41-cicd-kubernetes-por-servico.md) | CI/CD, qualidade, cobertura e deploy independente | To Do |
| [CARD-42](CARD-42-observabilidade-distribuida.md) | Observabilidade do fluxo entre serviços | To Do |
| [CARD-43](CARD-43-documentacao-e-entrega.md) | Swagger/Postman, vídeo e PDF final | To Do |

## Critérios globais da Fase 4

Diagrama de componentes: [Componentes — Fase 4](../../diagramas/diagrama-componentes-fase4.md).

Diagramas da Saga: [sequência](../../diagramas/diagrama-sequencia-saga-fase4.md) e [estados](../../diagramas/diagrama-estados-saga-os-fase4.md).

- Três ou mais microsserviços independentes, cada um com repositório, infraestrutura e banco próprios.
- Ao menos um banco relacional e um não relacional; CARD-35 registra se a leitura adotada é por solução ou por serviço e justifica a distribuição.
- REST síncrono onde necessário e mensageria assíncrona para integração desacoplada; nenhum acesso ao banco de outro serviço.
- Saga coordenando o fluxo crítico, com estado persistido, idempotência e compensação validada em falha.
- Mercado Pago integrado em sandbox, incluindo notificação/webhook e validação de resultado.
- Testes unitários em todos os serviços, pelo menos um fluxo BDD integrado, cobertura mínima de 80% por serviço e análise de qualidade no CI.
- Pipeline independente por serviço com build, testes, qualidade e deploy automatizado em Kubernetes.
- Reuso e extensão da observabilidade da Fase 3.
- Evidências por serviço, documentação e APIs atualizadas, vídeo de até 15 minutos e PDF final no portal.

---

## Fora do primeiro incremento

Não criar microsserviço adicional sem necessidade demonstrada; não trocar de nuvem por preferência; não implementar otimização de escala, multi-região ou mecanismos avançados de alta disponibilidade antes que o fluxo rubricado esteja funcionando. Esses itens podem ser reabertos mediante requisito ou ADR específico.