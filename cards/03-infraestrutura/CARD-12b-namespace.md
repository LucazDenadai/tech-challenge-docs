# CARD-12b — Namespace

**Depende de:** CARD-12a  
**Bloqueia:** CARD-12c

---

## Contexto

Namespace isola logicamente os recursos do sistema dentro do cluster. Todos os recursos do projeto ficam em `oficina-mecanica`, separados do `default` e do `kube-system`.

---

## Tarefas

**1. Criar o manifesto**

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: oficina-mecanica
```

**2. Aplicar**

```powershell
kubectl apply -f k8s/namespace.yaml
```

---

## Validação

```powershell
kubectl get namespace oficina-mecanica
# STATUS deve ser Active

kubectl get all -n oficina-mecanica
# "No resources found" — correto, namespace existe mas ainda vazio
```

---

## Entregável

- [ ] Namespace `oficina-mecanica` com status `Active`
