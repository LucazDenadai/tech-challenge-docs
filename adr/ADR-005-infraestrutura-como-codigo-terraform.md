# ADR-005 — Infraestrutura como Código com Terraform (Kind local)

**Status:** Superseded por [ADR-009](ADR-009-migracao-aws-e-separacao-repositorios.md)
**Data:** 2026-06-10  
**Autores:** Time Tech Challenge — Fase 3

---

## Contexto

Com os manifestos Kubernetes definidos no ADR-003 e aplicados no CARD-12, o próximo passo é provisionar o cluster e o banco de dados de forma reproduzível via código. Qualquer pessoa do time (ou avaliador) precisa recriar o ambiente do zero com um único comando. Terraform é a ferramenta padrão de mercado para Infrastructure as Code (IaC) e foi escolhida como requisito do CARD-13.

A decisão central é: **onde rodar o cluster?** As opções são ambiente local (Kind via Docker) ou cloud pública (GKE, EKS, AKS).

---

## Decisão

Usar **Terraform com provider Kind** para provisionar o cluster Kubernetes localmente via Docker, e um **container PostgreSQL via provider Docker** para o banco de dados. Ambos os módulos são independentes e substituíveis.

### Por que Kind e não cloud

| Critério | Kind local | Cloud (GKE/EKS/AKS) |
|---|---|---|
| Custo | Gratuito | Free tier limitado; risco de cobrança |
| Pré-requisitos | Docker + Terraform | Conta cloud + CLI + credenciais |
| Tempo de provisionamento | ~30 segundos | 5–15 minutos |
| Demonstração offline | Sim | Não |
| Complexidade do Terraform | Baixa | Alta (VPC, IAM, subnets, roles) |
| Fidelidade ao ADR-003 | Total — mesmo cluster K8s real | Total |

Para um Tech Challenge avaliado localmente, Kind é a escolha óbvia. Em produção real, o módulo `cluster` seria substituído por um módulo GKE/EKS mantendo os mesmos outputs — essa portabilidade é um benefício explícito da estrutura modular.

### Estrutura de módulos

```
infra/
├── versions.tf               ← versões fixas dos providers
├── main.tf                   ← orquestra os módulos
├── variables.tf              ← variáveis com defaults documentados
├── outputs.tf                ← endpoint do cluster + connection string
├── terraform.tfvars.example  ← exemplo sem valores sensíveis
└── modules/
    ├── cluster/
    │   ├── main.tf           ← Kind cluster + namespaces + service account
    │   ├── variables.tf
    │   └── outputs.tf
    └── database/
        ├── main.tf           ← container PostgreSQL + schemas
        ├── variables.tf
        └── outputs.tf
```

Dois módulos com responsabilidades distintas: `cluster` cuida do Kubernetes (incluindo o namespace `observabilidade` previsto no CARD-16), `database` cuida do PostgreSQL. Cada módulo é substituível independentemente.

### Namespaces provisionados pelo módulo cluster

| Namespace | Propósito |
|---|---|
| `oficina-mecanica` | Serviços de aplicação (Atendimento, Estoque) |
| `observabilidade` | Stack OTel: Jaeger, Prometheus, Grafana (CARD-16) |

### Providers utilizados

| Provider | Versão | Função |
|---|---|---|
| `tehcyx/kind` | ~> 0.4 | Criar/destruir cluster Kind via Docker |
| `hashicorp/kubernetes` | ~> 2.x | Criar namespaces e service account no cluster |
| `kreuzwerker/docker` | ~> 3.x | Criar container PostgreSQL |
| `hashicorp/null` | ~> 3.x | Executar scripts SQL de inicialização de schema |

---

## Alternativas consideradas

### Alternativa 1: GKE Autopilot (Google Cloud free tier)

**Prós:** ambiente mais próximo de produção, cluster K8s gerenciado, $300 de crédito em conta nova.

**Contras:** requer conta GCP ativa e credenciais configuradas, provisionamento demora ~10 min, não funciona sem internet, risco de cobrança acidental se o free tier for ultrapassado, Terraform muito mais complexo (VPC, IAM, subnets).

**Por que não:** inviabiliza avaliação offline e adiciona atrito desnecessário para um desafio acadêmico.

---

### Alternativa 2: LocalStack (AWS fake local)

Simula serviços AWS (EKS, RDS) localmente via Docker. Terraform usa o provider AWS apontando para `localhost`.

**Prós:** demonstra familiaridade com provider AWS, funciona offline.

**Contras:** LocalStack gratuito tem limitações no EKS simulado; a simulação não é 100% fiel; adiciona uma camada extra de complexidade sem benefício real para o desafio.

**Por que não:** Kind provisiona um cluster K8s real com menos complexidade e mais fidelidade.

---

### Alternativa 3: Minikube em vez de Kind

**Prós:** mais difundido, tem dashboard integrado.

**Contras:** o provider Terraform para Minikube é menos maduro que o provider Kind; Kind é o padrão de facto para clusters K8s em CI/CD local.

**Por que não:** Kind tem melhor suporte no ecossistema Terraform e é mais leve em recursos.

---

### Alternativa 4: PostgreSQL dentro do cluster K8s via provider kubernetes

Provisionar o PostgreSQL diretamente como `Deployment` K8s pelo provider `kubernetes` do Terraform, sem o provider Docker.

**Prós:** tudo dentro do cluster, sem dependência do provider Docker.

**Contras:** o provider `kubernetes` não tem recurso nativo para executar scripts SQL — exigiria `null_resource` + `kubectl exec`, mais frágil e menos legível.

**Por que não:** o provider Docker é mais direto para este caso e mantém a separação de responsabilidades (cluster vs banco).

---

## Consequências

### Positivas
- Ambiente recriável do zero em ~1 minuto com `terraform apply`
- `terraform destroy` limpa tudo sem rastros
- Estrutura modular permite trocar Kind por cloud sem reescrever o módulo database
- Versões fixas nos providers garantem reprodutibilidade entre máquinas
- Namespace `observabilidade` provisionado junto com o cluster, sem dependência extra no CARD-16

### Negativas e mitigações

| Risco | Mitigação |
|---|---|
| Docker precisa estar rodando antes do `terraform apply` | Documentado como pré-requisito no README de `/infra` |
| `terraform.tfstate` contém dados sensíveis | Adicionado ao `.gitignore`; nunca commitado |
| Dados do cluster são efêmeros (perdidos no `destroy`) | Comportamento esperado para ambiente de dev/demo |
| Provider `tehcyx/kind` é de terceiro (não HashiCorp) | Versão fixada com hash verificável; risco aceitável para demo |

---

## Referências

- [Terraform — Provider Kind (tehcyx)](https://registry.terraform.io/providers/tehcyx/kind/latest/docs)
- [Terraform — Provider Kubernetes](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs)
- [Terraform — Provider Docker (kreuzwerker)](https://registry.terraform.io/providers/kreuzwerker/docker/latest/docs)
- [Kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [ADR-003 — Arquitetura de Deploy no Kubernetes](ADR-003-arquitetura-kubernetes.md)
