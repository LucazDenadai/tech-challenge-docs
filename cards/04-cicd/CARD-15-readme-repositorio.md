# CARD-15 — README.md do repositório

**Tipo:** Documentação  
**Status:** To Do  
**Depende de:** CARD-13, CARD-14 (executar por último — documenta o que já existe)  
**Bloqueia:** nenhum

---

## Contexto

O PDF do desafio exige um README.md atualizado com conteúdo específico. Este card cobre exatamente o que foi pedido — nem mais, nem menos.

---

## Critérios de aceite

- [ ] README.md na raiz do repositório contém todas as seções abaixo
- [ ] Desenho de arquitetura incluído (pode ser diagrama ASCII ou imagem em `/docs/assets/`)
- [ ] Instruções de execução local funcionam do zero em uma máquina limpa
- [ ] Instruções de deploy K8s funcionam seguindo os passos descritos
- [ ] Instruções de Terraform funcionam seguindo os passos descritos
- [ ] Link da collection de APIs presente (Swagger ou Postman)
- [ ] Link do vídeo presente (YouTube ou Vimeo, até 15 minutos)

---

## Seções obrigatórias

### 1. Descrição da solução
- O que o sistema faz
- Objetivos desta fase (escalabilidade, resiliência, CI/CD)
- Decisões arquiteturais principais (link para ADR-001)

### 2. Desenho da arquitetura
Três subseções obrigatórias pelo PDF:

**Componentes da aplicação:**
```
- oficina-atendimento: OS, clientes, veículos, catálogo, auth
- oficina-estoque: peças, movimentações, consumer RabbitMQ
- RabbitMQ: mensageria entre os dois serviços
- PostgreSQL: banco compartilhado com schemas separados
```

**Infraestrutura provisionada:**
```
- Cluster Kubernetes (Kind local ou cloud)
- 2 Deployments com HPA (Atendimento + Estoque)
- StatefulSet RabbitMQ com PVC
- Deployment PostgreSQL com PVC
- ConfigMaps e Secrets por serviço
```

**Fluxo de deploy:**
```
Push em main
  → GitHub Actions CI (build + testes)
  → GitHub Actions CD (docker build + push)
  → kubectl apply manifestos K8s
  → kubectl rollout status (aguarda pods saudáveis)
```

### 3. Execução local

```bash
# Pré-requisitos: Docker, Docker Compose

git clone <repo>
cd Tech-challenge
cp .env.example .env
# editar .env com suas credenciais

docker-compose up --build

# Atendimento: http://localhost:8080/swagger
# Estoque:     http://localhost:8081/swagger
# RabbitMQ:    http://localhost:15672 (guest/guest)
```

### 4. Deploy em Kubernetes

> **Nota sobre o CI/CD:** o job de deploy do pipeline roda em self-hosted runner (ADR-006).
> O histórico de execuções está disponível no GitHub Actions. Para re-executar o deploy
> manualmente, siga os passos abaixo com `kubectl` apontando para o cluster local.

```bash
# Pré-requisitos: kubectl configurado apontando para o cluster

# Criar namespace
kubectl apply -f k8s/namespace.yaml

# Criar secrets (substituir os valores)
kubectl create secret generic atendimento-secrets \
  --from-literal=JWT_KEY=<valor> \
  --from-literal=DB_PASSWORD=<valor> \
  --from-literal=SMTP_PASSWORD=<valor> \
  -n oficina-mecanica

# Aplicar todos os manifestos
kubectl apply -f k8s/ -n oficina-mecanica

# Verificar pods
kubectl get pods -n oficina-mecanica

# Verificar HPA
kubectl get hpa -n oficina-mecanica
```

### 5. Provisionamento com Terraform

```bash
# Pré-requisitos: Terraform >= 1.6, Docker (para Kind)

cd infra
terraform init
terraform plan
terraform apply

# Outputs: endpoint do cluster, connection string do banco
```

### 6. Collection de APIs
Link para Swagger ou Postman collection.

### 7. Vídeo demonstrativo
Link YouTube ou Vimeo (até 15 minutos) demonstrando:
- Deploy da aplicação
- Execução do CI/CD
- Consumo das APIs
- Escalabilidade automática (HPA)

---

## Passos

1. Escrever todas as seções no README.md da raiz
2. Criar diagrama de arquitetura (ASCII ou imagem) e referenciar
3. Testar as instruções de execução local do zero
4. Testar as instruções de deploy K8s
5. Adicionar link do Swagger (gerado automaticamente pela API)
6. Adicionar link do vídeo após gravação
