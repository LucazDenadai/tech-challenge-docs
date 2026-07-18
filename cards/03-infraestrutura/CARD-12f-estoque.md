# CARD-12f — Estoque

**Depende de:** CARD-12c, CARD-12d  
**Bloqueia:** CARD-12g

---

## Contexto

Mesmo padrão do CARD-12e. O Estoque não precisa de JWT nem SMTP — apenas banco e RabbitMQ.

---

## Tarefas

**1. Build e carga da imagem no Kind**

```powershell
docker build -t oficina-estoque:latest `
  -f src/Estoque/OficinaMecanica.Estoque.API/Dockerfile .

kind load docker-image oficina-estoque:latest --name oficina
```

**2. ConfigMap**

```yaml
# k8s/estoque/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: estoque-config
  namespace: oficina-mecanica
data:
  ASPNETCORE_ENVIRONMENT: Production
  ASPNETCORE_URLS: http://+:8080
  RabbitMq__Enabled: "true"
  RabbitMq__Host: rabbitmq-svc
```

**3. Secret**

```yaml
# k8s/estoque/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: estoque-secrets
  namespace: oficina-mecanica
type: Opaque
data:
  DB_PASSWORD: cG9zdGdyZXM=   # "postgres" em base64
```

**4. Deployment**

```yaml
# k8s/estoque/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: estoque
  namespace: oficina-mecanica
spec:
  replicas: 2
  selector:
    matchLabels:
      app: estoque
  template:
    metadata:
      labels:
        app: estoque
    spec:
      containers:
        - name: estoque
          image: oficina-estoque:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: estoque-config
            - secretRef:
                name: estoque-secrets
          env:
            - name: ConnectionStrings__DefaultConnection
              value: "Host=postgres-svc;Port=5432;Database=oficina;Username=postgres;Password=$(DB_PASSWORD)"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

**5. Service**

```yaml
# k8s/estoque/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: estoque-svc
  namespace: oficina-mecanica
spec:
  selector:
    app: estoque
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30081
  type: NodePort
```

**6. HPA**

```yaml
# k8s/estoque/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: estoque-hpa
  namespace: oficina-mecanica
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: estoque
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

**7. Aplicar**

```powershell
kubectl apply -f k8s/estoque/
kubectl get pods -n oficina-mecanica -w
```

---

## Validação

```powershell
kubectl get pods -n oficina-mecanica
# estoque-xxx   1/1   Running
# estoque-yyy   1/1   Running

curl http://localhost:30081/health
```

---

## Entregável

- [ ] 2 Pods `estoque-*` em `Running`
- [ ] `GET localhost:30081/health` retorna 200
- [ ] HPA `estoque-hpa` visível
