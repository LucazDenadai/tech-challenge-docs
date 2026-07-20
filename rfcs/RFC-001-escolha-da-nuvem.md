# RFC-001 — Escolha da nuvem

**Status:** Aceito
**Data:** 2026-07-18
**Autores:** Time Tech Challenge — Fase 3
**ADR relacionado:** [ADR-009](../adr/ADR-009-migracao-aws-e-separacao-repositorios.md), [ADR-010](../adr/ADR-010-sizing-e-regiao-aws.md)

---

## Contexto

A Fase 3 exige infraestrutura corporativa: API Gateway, Function Serverless para autenticação, Banco de Dados Gerenciado e Cluster Kubernetes com escalabilidade, todos provisionados via Terraform com deploy automático. A Fase 2 rodava em Kind local (ADR-005) justamente para evitar essa decisão — sem requisito de nuvem pública, custo zero era possível. A Fase 3 remove essa opção: nenhum dos requisitos obrigatórios existe em ambiente local.

É preciso escolher um provedor de nuvem entre os três principais (AWS, Azure, GCP), todos capazes de atender aos requisitos técnicos do desafio.

---

## Proposta

Adotar **AWS** como provedor único para toda a infraestrutura da Fase 3.

| Requisito do desafio | Serviço AWS |
|---|---|
| API Gateway | Amazon API Gateway |
| Function Serverless (autenticação) | AWS Lambda (.NET) |
| Banco de Dados Gerenciado | Amazon RDS (PostgreSQL) |
| Cluster Kubernetes com escalabilidade | Amazon EKS + HPA |
| Terraform | Provider `hashicorp/aws` + módulos `terraform-aws-modules/vpc`, `/eks` |

### Critérios de decisão

1. **Cobertura direta dos requisitos**: os 5 requisitos obrigatórios de infraestrutura têm correspondência 1:1 com serviços AWS nativos, sem necessidade de composição ou workarounds.
2. **Familiaridade do time**: maior experiência prévia com AWS do que com Azure/GCP, reduzindo risco de erro de configuração sob prazo apertado.
3. **Volume de documentação e módulos da comunidade**: `terraform-aws-modules/eks` e `/vpc` são amplamente usados e documentados, reduzindo a superfície de código próprio a escrever e manter (ver ADR-010).
4. **Integração nativa Lambda + API Gateway**: é o par mais maduro e mais documentado entre as três nuvens para o padrão "autenticação via function serverless na borda".

---

## Alternativas consideradas

### Alternativa 1: Azure (API Management + Functions + AKS + Azure Database for PostgreSQL)

**Prós:** créditos de estudante tipicamente mais generosos; Azure Functions tem suporte de primeira classe a .NET (mesmo runtime do restante do projeto).

**Contras:** menor familiaridade do time; menos módulos Terraform maduros da comunidade para AKS comparado a EKS.

**Por que não:** a vantagem de crédito de estudante não compensa o risco de configuração incorreta sob prazo apertado, dado a menor experiência prévia do time com a plataforma.

### Alternativa 2: GCP (API Gateway + Cloud Functions + GKE Autopilot + Cloud SQL)

**Prós:** GKE Autopilot simplifica a operação do cluster (não é preciso gerenciar node groups manualmente); GCP costuma ter free tier mais generoso para novas contas.

**Contras:** GKE Autopilot abstrai decisões de sizing que o desafio pede para serem explícitas e documentadas (ex: tamanho de nós); menor volume de exemplos para a combinação API Gateway + Cloud Functions especificamente para autenticação via CPF/JWT.

**Por que não:** o desafio valoriza decisões de infraestrutura explícitas e justificadas (ADRs) — a automação do Autopilot reduziria o que há para decidir e documentar sobre sizing.

### Alternativa 3: Multi-cloud (ex: Lambda na AWS, cluster em outra nuvem)

**Prós:** nenhum benefício real identificado para o escopo deste desafio.

**Contras:** complexidade de rede/autenticação cross-cloud sem ganho correspondente; nenhum requisito do desafio pede ou se beneficia de multi-cloud.

**Por que não:** complexidade pura sem justificativa técnica ou de negócio.

---

## Decisão

AWS, com todos os 4 serviços (API Gateway, Lambda, RDS, EKS) na região `us-east-1` (ver ADR-010 para a justificativa de região e sizing).

## Consequências

Ver seção "Consequências" do [ADR-009](../adr/ADR-009-migracao-aws-e-separacao-repositorios.md), que registra a decisão formal e suas mitigações de risco.
