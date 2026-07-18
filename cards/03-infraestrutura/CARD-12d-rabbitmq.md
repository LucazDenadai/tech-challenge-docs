# CARD-12d — RabbitMQ

**Depende de:** CARD-12b  
**Bloqueia:** CARD-12e

---

## Contexto

RabbitMQ usa `StatefulSet` em vez de `Deployment` porque precisa de nome de Pod previsível (`rabbitmq-0`) para clustering futuro. Um `Deployment` gera nomes aleatórios; um `StatefulSet` garante identidade estável.

---

## Tarefas

**1. PVC**

```yaml
# k8s/rabbitmq/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rabbitmq-data
  namespace: oficina-mecanica
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**2. StatefulSet**

```yaml
# k8s/rabbitmq/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
  namespace: oficina-mecanica
spec:
  serviceName: rabbitmq-svc
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
        - name: rabbitmq
          image: rabbitmq:3.13-management-alpine
          ports:
            - containerPort: 5672
            - containerPort: 15672
          env:
            - name: RABBITMQ_DEFAULT_USER
              value: guest
            - name: RABBITMQ_DEFAULT_PASS
              value: guest
          volumeMounts:
            - name: data
              mountPath: /var/lib/rabbitmq
          readinessProbe:
            exec:
              command: ["rabbitmq-diagnostics", "ping"]
            initialDelaySeconds: 20
            periodSeconds: 10
          livenessProbe:
            exec:
              command: ["rabbitmq-diagnostics", "ping"]
            initialDelaySeconds: 60
            periodSeconds: 15
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: rabbitmq-data
```

**3. Service**

```yaml
# k8s/rabbitmq/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-svc
  namespace: oficina-mecanica
spec:
  selector:
    app: rabbitmq
  ports:
    - name: amqp
      port: 5672
      targetPort: 5672
    - name: management
      port: 15672
      targetPort: 15672
  type: ClusterIP
```

**4. Aplicar**

```powershell
kubectl apply -f k8s/rabbitmq/
kubectl get pods -n oficina-mecanica -w   # aguardar rabbitmq-0 Running
```

**5. Acessar a Management UI para confirmar**

```powershell
kubectl port-forward svc/rabbitmq-svc 15672:15672 -n oficina-mecanica
# Abrir: http://localhost:15672  (guest / guest)
```

---

## Validação

```powershell
kubectl get pods -n oficina-mecanica
# rabbitmq-0   1/1   Running

kubectl get pvc -n oficina-mecanica
# rabbitmq-data   Bound
```

---

## Entregável

- [ ] Pod `rabbitmq-0` em `Running`
- [ ] PVC `rabbitmq-data` em `Bound`
- [ ] Management UI acessível em `localhost:15672` via port-forward
