d# CARD-13 — Terraform: infraestrutura como código

**Tipo:** Infra  
**Status:** To Do  
**Depende de:** CARD-12  
**Bloqueia:** CARD-14 (CI/CD)  
**Decisão arquitetural:** [ADR-005](../../adr/ADR-005-infraestrutura-como-codigo-terraform.md)

---

## Contexto

Criar scripts Terraform para provisionar o cluster Kubernetes (Kind local via Docker) e o banco de dados PostgreSQL. O objetivo é que qualquer pessoa consiga recriar o ambiente do zero com `terraform apply`. A estrutura modular permite substituir Kind por cloud (GKE/EKS) sem alterar o módulo de banco.

O módulo `cluster` provisiona dois namespaces: `oficina-mecanica` (serviços de aplicação) e `observabilidade` (reservado para o CARD-16 — Jaeger, Prometheus, Grafana).

---

## Critérios de aceite

- [ ] `terraform init` executa sem erro
- [ ] `terraform plan` mostra os recursos a criar sem erros
- [ ] `terraform apply` provisiona cluster Kind + container PostgreSQL do zero
- [ ] Namespace `oficina-mecanica` criado no cluster
- [ ] Namespace `observabilidade` criado no cluster
- [ ] Container PostgreSQL acessível com schemas `atendimento` e `estoque` criados
- [ ] Outputs documentados: kubeconfig path + connection string do banco
- [ ] `terraform destroy` desfaz tudo sem erro
- [ ] `terraform.tfstate` adicionado ao `.gitignore`
- [ ] `README.md` em `/infra` com pré-requisitos, comandos e outputs

---

## Pré-requisitos do ambiente

- Docker rodando
- Terraform >= 1.6 instalado
- Kind instalado (usado internamente pelo provider)

---

## Estrutura de arquivos

```
infra/
├── versions.tf               ← versões fixas dos providers
├── main.tf                   ← orquestra os módulos
├── variables.tf              ← variáveis com defaults documentados
├── outputs.tf                ← kubeconfig path + connection string
├── terraform.tfvars.example  ← exemplo de valores sem dados sensíveis
└── modules/
    ├── cluster/
    │   ├── main.tf           ← Kind cluster + namespaces + service account
    │   ├── variables.tf
    │   └── outputs.tf
    └── database/
        ├── main.tf           ← container PostgreSQL + scripts de schema
        ├── variables.tf
        └── outputs.tf
```

---

## Recursos provisionados

### Módulo `cluster`
- Cluster Kind (`oficina-mecanica`)
- Namespace `oficina-mecanica`
- Namespace `observabilidade`
- ServiceAccount com permissões mínimas

### Módulo `database`
- Container Docker com `postgres:16`
- Banco `oficinamecanica`
- Schemas `atendimento` e `estoque`
- Usuário e senha via variáveis (não hardcoded)

---

## Providers utilizados

| Provider | Versão | Função |
|---|---|---|
| `tehcyx/kind` | ~> 0.4 | Criar cluster Kind via Docker |
| `hashicorp/kubernetes` | ~> 2.x | Criar namespaces e service account |
| `kreuzwerker/docker` | ~> 3.x | Criar container PostgreSQL |
| `hashicorp/null` | ~> 3.x | Executar scripts SQL de schema |

---

## Exemplo de uso

```bash
cd infra

# Primeira vez
terraform init

# Ver o que será criado (seguro, não altera nada)
terraform plan

# Provisionar tudo
terraform apply

# Destruir tudo
terraform destroy
```

---

## Outputs esperados

| Output | Descrição |
|---|---|
| `cluster_endpoint` | Endereço do API server do Kind |
| `kubeconfig_path` | Caminho do kubeconfig gerado |
| `postgres_connection_string` | Connection string para uso nas aplicações |

---

## README obrigatório em /infra

Deve conter:
1. Pré-requisitos (Docker, Terraform, Kind)
2. Como inicializar (`terraform init`)
3. Como provisionar (`terraform plan` → `terraform apply`)
4. Outputs e como usá-los
5. Como destruir (`terraform destroy`)
6. O que **não** commitar (`terraform.tfstate`, `terraform.tfvars`)

---

## Passos

1. Criar estrutura de pastas em `/infra`
2. Criar `versions.tf` com versões fixas de todos os providers
3. Implementar `modules/cluster/` (Kind + namespaces)
4. Implementar `modules/database/` (PostgreSQL + schemas)
5. Criar `main.tf`, `variables.tf` e `outputs.tf` na raiz
6. Criar `terraform.tfvars.example`
7. Adicionar `terraform.tfstate*` e `terraform.tfvars` ao `.gitignore`
8. Validar ciclo completo: `init` → `plan` → `apply` → `destroy`
9. Escrever `README.md` em `/infra`
