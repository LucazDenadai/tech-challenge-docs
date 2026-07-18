# CARD-12e — Atendimento

**Depende de:** CARD-12c, CARD-12d  
**Bloqueia:** CARD-12g

---

## Contexto

Configurações não sensíveis (URLs, flags) ficam no ConfigMap. Credenciais (JWT, senha do banco) ficam no Secret. O Deployment injeta os dois como variáveis de ambiente. O Service tipo NodePort expõe o serviço em `localhost:30080` para teste com Postman.

Como ainda não há CI/CD, a imagem é buildada localmente e carregada diretamente no cluster Kind.

---

## Tarefas

**1. Build e carga da imagem no Kind**

```powershell
docker build -t oficina-atendimento:latest `
  -f src/Atendimento/OficinaMecanica.Atendimento.API/Dockerfile .

kind load docker-image oficina-atendimento:latest --name oficina
```

**2. ConfigMap**

```yaml
# k8s/atendimento/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: atendimento-config
  namespace: oficina-mecanica
data:
  ASPNETCORE_ENVIRONMENT: Production
  ASPNETCORE_URLS: http://+:8080
  RabbitMq__Enabled: "true"
  RabbitMq__Host: rabbitmq-svc
  EstoqueHttp__Enabled: "true"
  EstoqueServiceUrl: http://estoque-svc
  Jwt__Issuer: oficina-atendimento
  Jwt__Audience: oficina-atendimento-api
  Jwt__ExpiracaoMinutos: "60"
  Smtp__Host: smtp.gmail.com
  Smtp__Port: "587"
  Smtp__From: "Oficina Mecanica <oficina@exemplo.com>"
```

**3. Secret**

```yaml
# k8s/atendimento/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: atendimento-secrets
  namespace: oficina-mecanica
type: Opaque
data:
  # "placeholder-dev-key-minimum-32-chars-here!" em base64
  Jwt__Key: cGxhY2Vob2xkZXItZGV2LWtleS1taW5pbXVtLTMyLWNoYXJzLWhlcmUh
  # "postgres" em base64
  DB_PASSWORD: cG9zdGdyZXM=
  SMTP_PASSWORD: ""
```

**4. Deployment**

```yaml
# k8s/atendimento/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: atendimento
  namespace: oficina-mecanica
spec:
  replicas: 2
  selector:
    matchLabels:
      app: atendimento
  template:
    metadata:
      labels:
        app: atendimento
    spec:
      containers:
        - name: atendimento
          image: oficina-atendimento:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: atendimento-config
            - secretRef:
                name: atendimento-secrets
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
# k8s/atendimento/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: atendimento-svc
  namespace: oficina-mecanica
spec:
  selector:
    app: atendimento
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
  type: NodePort
```

**6. HPA**

```yaml
# k8s/atendimento/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: atendimento-hpa
  namespace: oficina-mecanica
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: atendimento
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
kubectl apply -f k8s/atendimento/
kubectl get pods -n oficina-mecanica -w
```

---

## Validação

```powershell
kubectl get pods -n oficina-mecanica
# atendimento-xxx   1/1   Running
# atendimento-yyy   1/1   Running

kubectl get hpa -n oficina-mecanica
# atendimento-hpa   ...   2/10 replicas

# Health check direto
curl http://localhost:30080/health
```

---

## Entregável

- [ ] 2 Pods `atendimento-*` em `Running`
- [ ] `GET localhost:30080/health` retorna 200
- [ ] HPA `atendimento-hpa` visível
