# ADR-009 — Migração para AWS e Separação em 4 Repositórios

**Status:** Aceito
**Data:** 2026-07-17
**Autores:** Time Tech Challenge — Fase 3

---

## Contexto

A Fase 3 exige nível de operação corporativa: API Gateway com autenticação via CPF emitida por uma Function Serverless, Cluster Kubernetes "com escalabilidade", Banco de Dados Gerenciado, e o projeto organizado em **4 repositórios separados**, cada um com CI/CD e deploy automático para a nuvem.

O ADR-005 decidiu Kind local + container Docker para banco, e o ADR-006 decidiu self-hosted runner — ambos justificados pelo desafio da Fase 2 aceitar cluster local e pela ausência de requisito de nuvem pública. A Fase 3 elimina essa margem: exige explicitamente "Cluster Kubernetes com escalabilidade" gerenciado, "Banco de Dados Gerenciado" e um API Gateway na nuvem, além de Lambda — nenhum desses existe em ambiente local. Os riscos que motivaram ficar local (custo, tempo de provisionamento, dependência de conta cloud) deixam de ser evitáveis.

---

## Decisão

### Nuvem: AWS

Adotar **AWS** como provedor. Critério de escolha: API Gateway + Lambda são serviços nativos e a combinação EKS + RDS + API Gateway + Lambda é o caminho mais direto e documentado para os 5 requisitos de infraestrutura obrigatória do desafio (API Gateway, Function Serverless, Banco Gerenciado, Cluster K8s escalável, Terraform).

| Requisito do desafio | Serviço AWS |
|---|---|
| API Gateway | Amazon API Gateway |
| Function Serverless (auth) | AWS Lambda (.NET) |
| Banco de Dados Gerenciado | Amazon RDS (PostgreSQL) |
| Cluster Kubernetes com escalabilidade | Amazon EKS + HPA (mantido do ADR-003) |
| Terraform | Provider `hashicorp/aws` |

Isso **supersede o ADR-005** (Kind local via provider `tehcyx/kind` deixa de se aplicar) e **supersede o ADR-006** (self-hosted runner deixa de ser necessário — o runner `ubuntu-latest` do GitHub Actions acessa a AWS pela internet, sem depender de rede local).

### Lambda: .NET/C#, não Node.js

A Function Serverless de autenticação (validar CPF, consultar cliente no RDS, emitir JWT) é escrita em **.NET/C#**, mesma stack do restante do projeto — evita introduzir uma segunda linguagem/runtime só para esse componente. O cold start do .NET on AWS Lambda (ARM64, runtime atual) fica na faixa de 200-400ms; irrelevante em custo de execução (billing por ms) e aceitável para uma função de autenticação chamada uma vez por login.

### Estrutura de 4 repositórios (+ 1 de documentação)

| # | Repositório | Conteúdo |
|---|---|---|
| 1 | `tech-challenge-lambda` | Function Serverless de autenticação (.NET) |
| 2 | `tech-challenge-infra-k8s` | Terraform: EKS, VPC, API Gateway |
| 3 | `tech-challenge-infra-db` | Terraform: RDS PostgreSQL |
| 4 | `Tech-challenge` (este repositório) | Monorepo com os microsserviços Atendimento + Estoque, manifests K8s da aplicação, CI/CD |

Os dois microsserviços (Atendimento e Estoque) permanecem no mesmo repositório de aplicação — o desafio pede um único repositório para "Aplicação principal executando em Kubernetes"; separar em repositórios distintos por microsserviço não é exigido e fragmentaria desnecessariamente o pipeline de CI/CD deste componente.

Além dos 4 repositórios exigidos, um **5º repositório `tech-challenge-docs`** centraliza ADRs, RFCs e diagramas de arquitetura de todo o projeto. Justificativa: ADRs de infraestrutura (ex: ADR-003, ADR-009) e ADRs de aplicação (ex: ADR-001, ADR-007) documentam decisões que atravessam múltiplos dos 4 repositórios — centralizá-los evita que um avaliador precise reconstruir o histórico de decisões pulando entre repositórios, e evita duplicar ADRs em mais de um lugar. Cada um dos 4 repositórios exigidos mantém seu próprio `README.md` (autocontido, conforme entregável) e linka para `tech-challenge-docs` na seção de documentação arquitetural.

Cada repositório tem seu próprio pipeline de CI/CD, branch `main`/`master` protegida (sem commit direto), PR obrigatório para merge, e deploy automático para AWS a partir das branches de homologação/produção.

---

## Alternativas consideradas

### Alternativa 1: Manter Kind local e simular os requisitos de nuvem

Continuar com Kind + Docker, documentando os serviços de nuvem (API Gateway, Lambda, RDS) apenas como diagrama, sem implementação real.

**Por que não:** o desafio exige "deploy automático para a nuvem" e "links para os deploys ativos" como entregável — documentação sem implementação não atende ao requisito obrigatório.

### Alternativa 2: Azure ou GCP

Azure (API Management + Functions + AKS) ou GCP (API Gateway + Cloud Functions + GKE) atendem aos mesmos requisitos.

**Por que não escolhemos:** preferência do time por AWS, maior familiaridade e volume de documentação/exemplos para a combinação API Gateway + Lambda + EKS + RDS.

### Alternativa 3: Repositório por microsserviço (Atendimento e Estoque separados)

Separar cada microsserviço em seu próprio repositório, totalizando 5 repositórios.

**Por que não:** o desafio pede explicitamente 4 repositórios, com "Aplicação principal" como item único. Separar os microsserviços quebraria essa contagem sem ganho — os dois já evoluem em conjunto e compartilham o mesmo banco (ADR-007).

---

## Consequências

### Positivas
- Atende integralmente aos requisitos obrigatórios de infraestrutura da Fase 3
- Runner `ubuntu-latest` padrão do GitHub Actions volta a ser suficiente — elimina a dependência operacional do ADR-006 (máquina local sempre ligada)
- Estrutura modular do Terraform (ADR-005) é reaproveitada — módulo `cluster` e `database` são substituídos por equivalentes AWS, mantendo a mesma separação de responsabilidades
- Repositórios separados permitem pipelines e permissões independentes por componente

### Negativas e mitigações

| Risco | Mitigação |
|---|---|
| Custo de infraestrutura AWS (EKS, RDS, API Gateway não são gratuitos) | Usar tipos de instância mínimos (t3.small/micro), destruir recursos fora de janelas de demonstração/avaliação via `terraform destroy` |
| Tempo de provisionamento maior (EKS leva ~10-15 min) | Documentado no README; não impacta desenvolvimento local, que continua via Docker Compose |
| Cold start da Lambda .NET (~200-400ms) | Aceitável para fluxo de login; sem otimização adicional necessária no escopo deste desafio |
| Credenciais AWS em CI/CD | Usar OIDC do GitHub Actions para assumir role na AWS, evitando secrets de long-lived access keys |

---

## Referências

- [ADR-003 — Arquitetura de Deploy no Kubernetes](ADR-003-arquitetura-kubernetes.md)
- [ADR-005 — Infraestrutura como Código com Terraform (Kind local)](ADR-005-infraestrutura-como-codigo-terraform.md) — superseded por este ADR
- [ADR-006 — Self-Hosted Runner para Deploy no CI/CD](ADR-006-self-hosted-runner-cicd.md) — superseded por este ADR
- [AWS API Gateway — Documentação](https://docs.aws.amazon.com/apigateway/)
- [AWS Lambda — .NET](https://docs.aws.amazon.com/lambda/latest/dg/lambda-dotnet.html)
- [Terraform — Provider AWS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
