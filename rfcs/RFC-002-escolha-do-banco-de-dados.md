# RFC-002 — Escolha do banco de dados gerenciado

**Status:** Aceito
**Data:** 2026-07-18
**Autores:** Time Tech Challenge — Fase 3
**ADR relacionado:** [ADR-007](../adr/ADR-007-banco-compartilhado-schemas-separados.md), [ADR-009](../adr/ADR-009-migracao-aws-e-separacao-repositorios.md), [ADR-010](../adr/ADR-010-sizing-e-regiao-aws.md)

---

## Contexto

Esta RFC trata de uma decisão distinta da já registrada no [ADR-007](../adr/ADR-007-banco-compartilhado-schemas-separados.md): o ADR-007 decidiu a **topologia lógica** dos dados (uma instância, schemas `atendimento` e `estoque` separados, em vez de bancos isolados por serviço). Essa decisão permanece válida e não é reaberta aqui.

O que muda na Fase 3 é a **infraestrutura física**: o desafio exige "Banco de Dados Gerenciado (PostgreSQL, MySQL, SQL Server, etc.)" como requisito obrigatório — substituindo o container PostgreSQL rodando via Docker (ADR-005) por um serviço gerenciado de nuvem.

---

## Proposta

Adotar **Amazon RDS para PostgreSQL** como motor de banco de dados gerenciado, mantendo a mesma engine (PostgreSQL 16) já usada desde a Fase 1, e a mesma topologia de schemas do ADR-007.

### Critérios de decisão

1. **Continuidade de engine**: manter PostgreSQL evita reescrever migrations do EF Core, revalidar comportamento de tipos de dados e queries já testadas nas Fases 1 e 2.
2. **Coerência com o provedor de nuvem**: já decidido AWS ([RFC-001](RFC-001-escolha-da-nuvem.md)) — RDS é o serviço gerenciado nativo, com integração direta a VPC, security groups e Secrets Manager.
3. **Menor esforço de migração de dados**: como a engine não muda, migrar de container Docker para RDS é um `pg_dump`/`pg_restore` ou simplesmente recriar o schema do zero (ambiente de desenvolvimento/estudo, sem dados de produção reais a preservar).
4. **Custo**: `db.t3.micro` é a menor instância viável para RDS PostgreSQL, compatível com o sizing decidido no [ADR-010](../adr/ADR-010-sizing-e-regiao-aws.md).

---

## Alternativas consideradas

### Alternativa 1: Amazon Aurora PostgreSQL

**Prós:** melhor performance e escalabilidade horizontal de leitura; mais alinhado a cargas de produção reais.

**Contras:** custo mínimo mais alto que RDS PostgreSQL padrão (não há instância comparável ao `db.t3.micro` em custo); complexidade de configuração (cluster de réplicas) desnecessária para o volume de uso deste desafio.

**Por que não:** a escala e disponibilidade adicionais do Aurora não têm uso real neste contexto — é otimização para um problema (alta concorrência de leitura, failover automático complexo) que o desafio não tem.

### Alternativa 2: Trocar de engine (MySQL ou SQL Server gerenciado)

**Prós:** nenhum ganho técnico identificado — o desafio aceita qualquer um dos três (PostgreSQL, MySQL, SQL Server).

**Contras:** exigiria reescrever migrations do EF Core, revalidar tipos de dados e comportamento de queries já testados desde a Fase 1; sem justificativa técnica para o retrabalho.

**Por que não:** troca de engine sem ganho técnico é retrabalho puro — vai contra a decisão já registrada e testada desde a Fase 1.

### Alternativa 3: Manter PostgreSQL via container no próprio EKS (não gerenciado)

**Prós:** evita custo adicional de uma instância RDS separada; mantém a mesma abordagem operacional da Fase 2.

**Contras:** não atende ao requisito obrigatório explícito do desafio ("Banco de Dados Gerenciado"); perde os benefícios de um serviço gerenciado (backups automáticos, patching, monitoramento nativo via CloudWatch).

**Por que não:** viola requisito obrigatório do desafio — não é uma alternativa viável, apenas foi considerada para registro de que a opção existe.

---

## Decisão

Amazon RDS para PostgreSQL 16, instância `db.t3.micro`, na mesma VPC do cluster EKS (via [tech-challenge-infra-db](https://github.com/LucazDenadai/tech-challenge-infra-db)), com schemas `atendimento` e `estoque` conforme ADR-007.

## Consequências

Ver seção "Consequências" do [ADR-010](../adr/ADR-010-sizing-e-regiao-aws.md) para custo e mitigações de risco relacionadas ao sizing escolhido.
