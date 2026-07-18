# CARD-12c — PostgreSQL

**Depende de:** CARD-12b  
**Bloqueia:** CARD-12e

---

## Contexto

Containers são efêmeros — se o Pod morrer, os dados somem. O PersistentVolumeClaim (PVC) reserva um volume no cluster que sobrevive ao ciclo de vida do Pod. O Service dá um DNS estável (`postgres-svc`) para os outros serviços se conectarem.

---

## Tarefas

**1. PVC — reserva o disco**

```yaml
# k8s/postgres/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: oficina-mecanica
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

**2. Secret — credenciais**

> Valores em base64. Para gerar: `[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("sua-senha"))`

```yaml
# k8s/postgres/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: oficina-mecanica
type: Opaque
data:
  POSTGRES_PASSWORD: cG9zdGdyZXM=  # postgres
  POSTGRES_USER: cG9zdGdyZXM=       # postgres
  POSTGRES_DB: b2ZpY2luYQ==         # oficina
```

**3. Deployment**

```yaml
# k8s/postgres/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: oficina-mecanica
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgres-secret
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 30
            periodSeconds: 10
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
            claimName: postgres-data
```

**4. Service**

```yaml
# k8s/postgres/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: oficina-mecanica
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP
```

**5. Aplicar**

```powershell
kubectl apply -f k8s/postgres/
kubectl get pods -n oficina-mecanica -w   # aguardar Running
```

---

## Validação

```powershell
kubectl get pods -n oficina-mecanica
# postgres-xxx   1/1   Running

kubectl get pvc -n oficina-mecanica
# postgres-data   Bound

kubectl get svc -n oficina-mecanica
# postgres-svc   ClusterIP
```

---

## Entregável

- [ ] Pod `postgres-*` em `Running`
- [ ] PVC `postgres-data` em `Bound`
- [ ] Service `postgres-svc` visível
