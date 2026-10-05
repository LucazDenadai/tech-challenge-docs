# ADR-014 — Limites dos microsserviços da Fase 4

**Status:** Aceito
**Data:** 2026-10-05
**Autores:** Time Tech Challenge — Fase 4

---

## Contexto

A Fase 4 exige pelo menos três microsserviços independentes. Cada um deve ter repositório, infraestrutura e banco de dados próprios; serviços não podem acessar diretamente o banco de outro serviço. O enunciado sugere OS, Orçamento e Pagamento (Billing), e Execução e Produção como responsabilidades, mas não exige que cada exemplo seja um microsserviço isolado.

O projeto existente tem Atendimento e Estoque como serviços separados. Separar também Execução do Estoque criaria quatro serviços de negócio e mais infraestrutura, embora ambos operem sobre o fluxo físico da oficina. O time decidiu manter essas duas capacidades no mesmo limite de serviço para reduzir a complexidade operacional sem reduzir o mínimo exigido.

---

## Decisão

A arquitetura de negócio da Fase 4 terá três microsserviços:

| Microsserviço | Responsabilidades e propriedade dos dados |
|---|---|
| **OS** | Abertura da OS, estado geral, consulta de status e histórico da ordem. É o único responsável por gravar o estado da OS. |
| **Billing** | Orçamentos, aprovação, pagamentos e integração com Mercado Pago. É o único responsável por orçamento e estado de pagamento. |
| **Operações** | Estoque e execução/produção: catálogo e saldo de peças, movimentações, fila de execução, diagnóstico, reparos e progresso operacional. É o único responsável pelos dados de estoque e execução. |

Estoque e Execução serão capacidades coesas dentro do microsserviço **Operações**, com um único repositório de código e uma unidade de deploy. O limite interno entre as capacidades deve continuar explícito no código e no modelo de dados; juntá-las não autoriza Billing ou OS a acessar seus dados diretamente.

Cada um dos três microsserviços terá repositório próprio, definição de infraestrutura e banco de dados sob sua propriedade. A tecnologia e a topologia física/lógica exatas dos bancos, incluindo a alocação SQL/NoSQL exigida pelo enunciado, serão decididas em registro próprio antes da implementação. Nenhum serviço compartilhará tabelas ou credenciais de banco com outro.

Os serviços colaborarão por contratos versionados: REST síncrono somente quando uma resposta imediata for necessária e mensageria assíncrona para eventos de domínio e integração desacoplada. Billing não altera a OS diretamente: publica o resultado do pagamento, e o serviço OS decide e persiste a transição correspondente.

### Decisões anteriores afetadas

- **ADR-001:** a decisão de manter somente Atendimento e Estoque não atende ao mínimo de três microsserviços. Para a Fase 4, o mapa de capacidades passa a ser OS, Billing e Operações.
- **ADR-007:** a decisão de banco compartilhado com schemas separados é substituída pela propriedade de banco por microsserviço.
- **ADR-009:** a decisão de manter os dois serviços de negócio num único repositório é substituída pela exigência de um repositório por microsserviço de negócio. Os repositórios de plataforma/documentação continuam separados conforme necessário.
- **ADR-004:** permanece válida a regra de fonte única da verdade para peças; a propriedade passa a residir na capacidade de Estoque do microsserviço Operações.

As decisões acima são supersedidas apenas para a arquitetura-alvo da Fase 4; os registros históricos das fases anteriores não devem ser reescritos.

---

## Alternativas consideradas

### Manter Estoque e Execução em microsserviços diferentes

**Não escolhida:** aumentaria o número de serviços, bancos, pipelines e deploys sem ser necessária para cumprir o mínimo de três. Continua possível reconsiderar caso surja uma necessidade concreta de escala, autonomia de equipe ou ciclo de release independente.

### Manter a arquitetura de Fase 3 e adicionar apenas Billing

**Rejeitada:** resultaria em apenas três serviços se Lambda fosse contada, mas Lambda é uma função de autenticação e não substitui o limite de domínio de Execução requerido pelo desafio. Também manteria a decisão de banco compartilhado, em conflito com a Fase 4.

---

## Consequências

### Positivas

- Atende ao mínimo de três microsserviços de negócio sem criar um quarto serviço apenas para separar responsabilidades próximas.
- Mantém Estoque como fonte da verdade para peças e adiciona o fluxo de execução no mesmo serviço operacional.
- Define responsáveis exclusivos pelos dados de OS, cobrança e operações, permitindo evolução sem acesso cruzado a bancos.
- Permite reaproveitar gradualmente domínio, API e mensageria existentes, com migração explícita das fronteiras antigas.

### Negativas e riscos

| Risco | Mitigação |
|---|---|
| Operações pode crescer e acumular responsabilidades | Preservar módulos internos e contratos distintos para Estoque e Execução; reavaliar a fronteira somente com evidência de acoplamento ou escala independente. |
| A decomposição de Fase 3 exige migração e coordenação entre repositórios | Planejar extração incremental, contratos compatíveis e testes de regressão antes de remover os caminhos antigos. |
| A exigência SQL/NoSQL não esclarece se vale por serviço ou para o sistema como um todo | Resolver a interpretação e registrar a alocação em ADR/RFC antes de criar bancos e pipelines. |

---

## Referências

- [Enunciado Tech Challenge — Fase 4](../cards/06-fase4-microsservicos-saga/README.md)
- [ADR-001 — Arquitetura de microsserviços e mensageria](ADR-001-arquitetura-microservicos-mensageria.md)
- [ADR-004 — Estoque como fonte da verdade para peças](ADR-004-estoque-fonte-verdade-pecas.md)
- [ADR-007 — Banco compartilhado (supersedido para Fase 4)](ADR-007-banco-compartilhado-schemas-separados.md)
- [ADR-009 — Migração AWS e separação de repositórios (supersedido parcialmente para Fase 4)](ADR-009-migracao-aws-e-separacao-repositorios.md)