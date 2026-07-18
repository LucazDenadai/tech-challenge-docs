# CARD-12a — Cluster local com Minikube

**Depende de:** Docker Desktop rodando  
**Bloqueia:** CARD-12b

---

## Contexto

Minikube cria um cluster Kubernetes local dentro de um container Docker. É o ambiente de desenvolvimento local mais usado em cursos — os manifestos YAML são idênticos aos de um cluster real (EKS).

---

## Tarefas

**1. Instalar Minikube**

```powershell
winget install Kubernetes.minikube
# fechar e reabrir o terminal
minikube version
```

**2. Iniciar o cluster**

```powershell
minikube start --driver=docker --cpus=2 --memory=4096
```

Usa o Docker Desktop como backend — sem VM separada. Na primeira execução baixa a imagem base do K8s (~500MB).

**3. Verificar**

```powershell
kubectl config current-context   # deve retornar: minikube
kubectl get nodes                # 1 nó em Ready
```

---

## Diferença importante: imagens locais no Minikube

Imagens buildadas na sua máquina não ficam visíveis dentro do cluster automaticamente. A solução é apontar o Docker para buildar direto dentro do Minikube — usaremos isso nos CARDs 12e e 12f:

```powershell
# Executar antes de cada "docker build"
minikube docker-env | Invoke-Expression
```

---

## Comandos úteis

```powershell
minikube status      # estado do cluster
minikube stop        # pausar sem destruir
minikube delete      # destruir e recriar do zero
minikube dashboard   # painel web do K8s no browser
```

---

## Entregável

- [ ] `minikube version` sem erro
- [ ] `kubectl get nodes` mostra nó `minikube` em `Ready`
- [ ] `kubectl config current-context` retorna `minikube`
