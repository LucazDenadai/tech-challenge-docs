# CARD-12g — Validação end-to-end

**Depende de:** CARD-12e, CARD-12f

---

## Contexto

Verificar que o cluster inteiro funciona como esperado: todos os Pods saudáveis, HPA operacional e o fluxo completo da OS (abertura → finalização → baixa de estoque via RabbitMQ) funcionando dentro do cluster.

---

## Tarefas

**1. Instalar o metrics-server** (necessário para o HPA ler CPU/memória)

```powershell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Kind usa TLS auto-assinado — patch para o metrics-server aceitar
kubectl patch deployment metrics-server -n kube-system `
  --type=json `
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

**2. Verificar todos os Pods**

```powershell
kubectl get pods -n oficina-mecanica

# Saída esperada (todos 1/1 Running):
# atendimento-xxx   1/1   Running
# atendimento-yyy   1/1   Running
# estoque-xxx       1/1   Running
# estoque-yyy       1/1   Running
# postgres-xxx      1/1   Running
# rabbitmq-0        1/1   Running
```

**3. Verificar HPAs**

```powershell
kubectl get hpa -n oficina-mecanica
# MINPODS=2, MAXPODS=10, REPLICAS=2
```

**4. Verificar PVCs**

```powershell
kubectl get pvc -n oficina-mecanica
# postgres-data   Bound
# rabbitmq-data   Bound
```

**5. Fluxo completo via Postman**

Usar a collection `docs/oficina-mecanica.postman_collection.json` com as variáveis:
- `base_atendimento` = `http://localhost:30080`
- `base_estoque` = `http://localhost:30081`

Executar na ordem: Login → Criar Peça → Criar Cliente → Criar Veículo → Fluxo OS completo (grupos 07 e 09).

Após finalizar a OS, `GET /estoque/pecas/{{peca_id}}` deve mostrar `quantidadeEstoque` reduzido.

**6. Diagnóstico — comandos úteis se algo falhar**

```powershell
# Ver logs de um Pod
kubectl logs -n oficina-mecanica deployment/atendimento --tail=50

# Ver eventos do namespace (crashes, pull errors, etc.)
kubectl get events -n oficina-mecanica --sort-by='.lastTimestamp'

# Descrever um Pod com problema
kubectl describe pod <nome-do-pod> -n oficina-mecanica

# Abrir shell dentro de um Pod
kubectl exec -it <nome-do-pod> -n oficina-mecanica -- sh
```

---

## Validação

- [ ] Todos os 6 Pods em `Running`
- [ ] `kubectl get hpa` mostra métricas reais (não `<unknown>`)
- [ ] `GET localhost:30080/health` e `GET localhost:30081/health` retornam 200
- [ ] Fluxo completo Postman executado sem erros
- [ ] Estoque reduzido após finalizar OS (baixa via RabbitMQ funcionando no cluster)

---

## Entregável

- [ ] Print ou output de `kubectl get pods -n oficina-mecanica` com todos Running
- [ ] Fluxo end-to-end confirmado via Postman
