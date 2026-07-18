# ADR-007 — Banco de Dados Compartilhado com Schemas Separados

**Status:** Aceito  
**Data:** 2026-06-11  
**Autores:** Time Tech Challenge — Fase 2

---

## Contexto

O sistema possui dois microserviços com domínios distintos: **Atendimento** (ordens de serviço, clientes, veículos) e **Estoque** (peças, movimentações). Em arquiteturas de microserviços, a prática recomendada é que cada serviço possua seu próprio banco de dados isolado, evitando acoplamento de dados entre domínios.

Durante o design da infraestrutura (CARD-12 e CARD-13), avaliamos três opções para o armazenamento.

---

## Opções consideradas

### Opção A — Um banco por serviço (dois PostgreSQL)
Cada microserviço com sua própria instância de PostgreSQL, completamente isolada.

**Prós:**
- Isolamento total de dados e de falha
- Escala independente por serviço — o banco do Atendimento pode ser escalado sem afetar o Estoque
- Autonomia completa de schema e versão

**Contras:**
- Dois Deployments de PostgreSQL no Kubernetes, dois PVCs, dois conjuntos de secrets
- Dois Migration Jobs no pipeline de CI/CD
- Maior complexidade operacional para o ambiente de desenvolvimento local (Kind)

### Opção B — Um banco, dois databases
Uma instância PostgreSQL com dois databases separados (`oficina_atendimento` e `oficina_estoque`).

**Prós:**
- Isolamento lógico entre domínios
- Uma instância só para operar

**Contras:**
- Exige scripts adicionais de bootstrap para criação dos databases
- Sem ganho real de escala independente — ainda é uma instância compartilhada

### Opção C — Um banco, dois schemas (escolhida)
Uma instância PostgreSQL com dois schemas: `atendimento` e `estoque`. Cada serviço usa sua própria connection string com `SearchPath` apontando para seu schema.

**Prós:**
- Um único Deployment K8s, um PVC, um conjunto de secrets de banco
- Terraform provisiona com um único módulo de database
- O isolamento via `DbContext` por serviço impede acesso cruzado no nível de aplicação
- Simplicidade operacional compatível com o escopo do projeto

**Contras:**
- **O banco não escala independentemente por serviço** — escalar o Atendimento em IOPS significa escalar o banco inteiro, impactando o Estoque desnecessariamente
- Ponto único de falha: se o PostgreSQL cair, ambos os serviços são afetados
- Acoplamento de infraestrutura: deploys de schema de um serviço compartilham a mesma instância do outro

---

## Decisão

Adotar a **Opção C** — um PostgreSQL com dois schemas separados.

**Justificativa:** Para o escopo deste projeto, a escalabilidade da camada de aplicação é garantida pelo **Horizontal Pod Autoscaler (HPA)** configurado nos Deployments Kubernetes (CARD-12, ADR-003). O gargalo esperado em picos de carga é processamento de requisições, não IOPS de banco — o que o HPA resolve horizontalmente sem exigir escala independente do banco.

O isolamento entre os domínios é garantido em código por:
- Connection strings distintas por serviço, cada uma apontando para seu schema
- `DbContext` independentes — sem referência cruzada de entidades via EF Core
- Migrations gerenciadas separadamente por serviço no seu próprio schema

---

## Tradeoff aceito

A decisão **sacrifica autonomia de escala do banco** em favor de **simplicidade operacional**. Isso é adequado para o volume simulado neste projeto e para um ambiente de desenvolvimento local com Kind.

Em um cenário produtivo com crescimento real de carga, este modelo precisaria ser revisado.

---

## Caminho de evolução (se necessário)

Caso o banco se torne gargalo, a migração para a **Opção A** (um PostgreSQL por serviço) pode ser feita sem alterar nenhuma linha de código de domínio ou aplicação:

1. Provisionar uma segunda instância de PostgreSQL (novo módulo Terraform)
2. Migrar os dados do schema `estoque` para o novo banco via `pg_dump / pg_restore`
3. Atualizar apenas a connection string do serviço Estoque (secret K8s + variável de ambiente)
4. Reexecutar as migrations do Estoque no novo banco
5. Remover o schema `estoque` do banco original

O isolamento via `DbContext` garante que nenhuma outra camada precise mudar — a troca é transparente para a aplicação.

---

## Referências

- ADR-001 — Arquitetura de Microserviços com Mensageria Assíncrona
- ADR-003 — Arquitetura Kubernetes (HPA e escala da aplicação)
- ADR-005 — Infraestrutura como Código com Terraform
