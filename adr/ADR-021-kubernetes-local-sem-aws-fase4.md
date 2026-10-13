# ADR-021 — Kubernetes da Fase 4 sem AWS: Kind no CI e na demonstração

**Status:** Aceito
**Data:** 2026-10-13
**Autores:** Time Tech Challenge — Fase 4
**Relaciona-se a:** [ADR-009](ADR-009-migracao-aws-e-separacao-repositorios.md), [ADR-015](ADR-015-ownership-e-infraestrutura-fase4.md) (substitui a plataforma EKS), [ADR-016](ADR-016-bancos-sql-nosql-fase4.md) (substitui "gerenciado"), [CARD-41](../cards/06-fase4-microsservicos-saga/CARD-41-cicd-kubernetes-por-servico.md), [CARD-41b](../cards/06-fase4-microsservicos-saga/CARD-41b-kubernetes-por-servico.md)

---

## Contexto

O ADR-015 previa reaproveitar o EKS da Fase 3 (`tech-challenge-infra-k8s`, Terraform) como plataforma compartilhada, e o ADR-016 previa um PostgreSQL gerenciado por serviço e uma tabela DynamoDB para Operações. Em 2026-10-13 o time ficou **sem conta AWS disponível** para a Fase 4.

O PDF da Fase 4 exige **deploy automatizado de cada microsserviço em Kubernetes** (requisito R-14 da [matriz](../cards/06-fase4-microsservicos-saga/matriz-requisitos.md)) e o vídeo mostra o deploy (E-07). AWS e Terraform não são requisitos do PDF (matriz, ambiguidades).

---

## Decisão

### Plataforma: Kind

- **No CI de cada serviço:** um cluster [Kind](https://kind.sigs.k8s.io/) efêmero sobe no runner, recebe a plataforma compartilhada e o serviço, e o job espera o rollout e o readiness. É o deploy automatizado do R-14: cada repositório implanta só o próprio serviço.
- **Na demonstração:** o mesmo Kind roda na máquina local, com os três serviços, para o vídeo.
- **Ownership mantido (ADR-015):** `tech-challenge-infra-k8s` passa a ser dono dos manifests da plataforma compartilhada no Kind (namespace, RabbitMQ, Jaeger) e do script que sobe o ambiente de demonstração. Cada repositório de serviço é dono dos próprios manifests (Kustomize em `k8s/`) e do próprio banco.

### Bancos

- Cada serviço roda **a própria instância PostgreSQL dentro do cluster**, versionada no repositório do serviço; continua exclusiva por serviço (ADR-016), mas não gerenciada.
- Operações usa o **DynamoDB Local** no cluster, a mesma imagem dos testes e do Compose.
- Os dados vivem enquanto o cluster existir. É uma configuração de demonstração, não de produção.

### Fora da Fase 4

- EKS, RDS, tabela DynamoDB gerenciada, Terraform dos bancos e API Gateway.
- **Lambda de autenticação por CPF** (roda na AWS Lambda): sem ela, o token de cliente da demonstração é gerado como no BDD (mesmo formato, mesma chave); a forma de apresentar isso no vídeo fica no CARD-43.
- O webhook do Mercado Pago continua possível pelo túnel da Cloudflare (CARD-39b).

---

## Alternativas consideradas

### EKS na janela de demonstração

**Inviável agora:** sem conta AWS. Volta a ser opção se houver conta; os manifests por serviço não mudam, só a plataforma.

### Outro Kubernetes gerenciado (GKE, AKS) com crédito gratuito

**Não escolhida:** exige nova conta e cartão, nova IaC e rede, para um requisito que o Kind já atende.

### Minikube ou k3d

**Não escolhida:** atenderiam igual; o Kind tem ação pronta no GitHub Actions e roda o mesmo cluster no CI e na máquina local.

---

## Consequências

### Positivas

- Deploy automatizado verificável em cada PR e merge, sem custo.
- O mesmo manifesto sobe no CI e na demonstração.

### Negativas e riscos

| Risco | Mitigação |
|---|---|
| Cluster efêmero: nada fica no ar entre as execuções | A demonstração sobe o ambiente com um script; o CI prova o rollout a cada mudança. |
| Banco no cluster perde dados ao apagar o cluster | Aceito para demonstração; seeds de cada serviço recriam os dados de exemplo. |
| Sem Lambda, sem API Gateway | Token de cliente no formato da Lambda (CARD-43); acesso aos serviços por porta exposta do Kind. |
| Avaliador esperar nuvem | O PDF não exige nuvem; a decisão e o motivo ficam neste ADR e no vídeo. |
