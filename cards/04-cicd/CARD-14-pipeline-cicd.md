# CARD-14 — Pipeline CI/CD (GitHub Actions)

**Tipo:** Infra  
**Status:** To Do  
**Depende de:** CARD-07, CARD-11, CARD-12, CARD-13, CARD-16  
**Bloqueia:** nenhum

---

## Contexto

Criar a pipeline GitHub Actions que executa build, testes, build de imagem Docker e deploy no cluster Kubernetes a cada push na branch `main`. O deploy só acontece se todos os testes passarem.

---

## Critérios de aceite

- [ ] Pipeline dispara em push para `main` e em pull requests
- [ ] PRs: apenas build + testes (sem deploy)
- [ ] Push em `main`: build + testes + docker build + push + deploy K8s
- [ ] Secrets configurados no repositório GitHub (não expostos nos arquivos)
- [ ] Deploy aplica os manifestos K8s e aguarda rollout
- [ ] **Migrations aplicadas antes do rollout dos pods** via Kubernetes Job
- [ ] Pipeline falha e não faz deploy se qualquer teste falhar
- [ ] Pipeline falha e não sobe os pods se a migration falhar
- [ ] Tempo total da pipeline abaixo de 10 minutos

---

## Estrutura de arquivos

```
.github/
└── workflows/
    └── dotnet.yml   # CI e CD como jobs sequenciais no mesmo arquivo
                     # build-and-test → docker → deploy
```

---

## ci.yml — Build e Testes

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Run unit tests
        run: dotnet test tests/**/UnitTests/*.csproj --no-build --configuration Release

      - name: Run integration tests
        run: dotnet test tests/**/IntegrationTests/*.csproj --no-build --configuration Release
```

---

## cd.yml — Docker + Deploy K8s

```yaml
name: CD

on:
  push:
    branches: [main]

needs: [build-and-test]   # só roda se CI passou

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build e push Atendimento
        uses: docker/build-push-action@v5
        with:
          context: .
          file: src/Atendimento/OficinaMecanica.Atendimento.API/Dockerfile
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/oficina-atendimento:${{ github.sha }}

      - name: Build e push Estoque
        uses: docker/build-push-action@v5
        with:
          context: .
          file: src/Estoque/OficinaMecanica.Estoque.API/Dockerfile
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/oficina-estoque:${{ github.sha }}

  deploy:
    runs-on: ubuntu-latest
    needs: [build-and-push]
    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}

      - name: Criar secrets no K8s
        run: |
          kubectl create secret generic atendimento-secrets \
            --from-literal=JWT_KEY=${{ secrets.JWT_KEY }} \
            --from-literal=DB_PASSWORD=${{ secrets.DB_PASSWORD }} \
            --from-literal=SMTP_PASSWORD=${{ secrets.SMTP_PASSWORD }} \
            --namespace=oficina-mecanica \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: Atualizar imagem no deployment
        run: |
          kubectl set image deployment/atendimento \
            atendimento=${{ secrets.DOCKER_USERNAME }}/oficina-atendimento:${{ github.sha }} \
            -n oficina-mecanica
          kubectl set image deployment/estoque \
            estoque=${{ secrets.DOCKER_USERNAME }}/oficina-estoque:${{ github.sha }} \
            -n oficina-mecanica

      - name: Aplicar manifestos K8s
        run: kubectl apply -f k8s/ -n oficina-mecanica

      - name: Aplicar stack de observabilidade
        run: kubectl apply -f k8s/observabilidade/ -n observabilidade

      - name: Aguardar rollout
        run: |
          kubectl rollout status deployment/atendimento -n oficina-mecanica
          kubectl rollout status deployment/estoque -n oficina-mecanica
```

---

## Secrets necessários no GitHub

| Secret | Descrição |
|---|---|
| `DOCKER_USERNAME` | Usuário do Docker Hub |
| `DOCKER_PASSWORD` | Token do Docker Hub |
| `KUBECONFIG` | Kubeconfig do cluster (base64) |
| `JWT_KEY` | Chave JWT (mínimo 32 chars) |
| `DB_PASSWORD` | Senha do PostgreSQL |
| `SMTP_PASSWORD` | Senha do email |

---

## Estratégia de migrations no K8s

Migrations são aplicadas via **Kubernetes Job** antes do rollout dos pods. O Job usa a mesma imagem do serviço e executa `dotnet ef database update`. Como as migrations do legado foram copiadas no CARD-05, o comando é idempotente — funciona em banco novo e existente.

### Arquivo: `k8s/atendimento/migration-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: atendimento-migration
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migration
          image: <DOCKER_USERNAME>/oficina-atendimento:<SHA>
          command: ["dotnet", "ef", "database", "update", "--no-build"]
          envFrom:
            - secretRef:
                name: atendimento-secrets
            - configMapRef:
                name: atendimento-config
  backoffLimit: 2
```

O mesmo padrão se aplica ao Estoque (`k8s/estoque/migration-job.yaml`).

### Ordem no cd.yml

```yaml
      - name: Aplicar migration Atendimento
        run: |
          kubectl apply -f k8s/atendimento/migration-job.yaml -n oficina-mecanica
          kubectl wait --for=condition=complete job/atendimento-migration \
            --timeout=120s -n oficina-mecanica

      - name: Aplicar migration Estoque
        run: |
          kubectl apply -f k8s/estoque/migration-job.yaml -n oficina-mecanica
          kubectl wait --for=condition=complete job/estoque-migration \
            --timeout=120s -n oficina-mecanica

      # Só chega aqui se ambos os Jobs completaram com sucesso
      - name: Aplicar manifestos K8s
        run: kubectl apply -f k8s/ -n oficina-mecanica

      - name: Aguardar rollout
        run: |
          kubectl rollout status deployment/atendimento -n oficina-mecanica
          kubectl rollout status deployment/estoque -n oficina-mecanica
```

---

## Passos

1. Criar `.github/workflows/ci.yml`
2. Criar `.github/workflows/cd.yml` com a ordem: migration Jobs → apply manifestos → aguardar rollout
3. Criar `k8s/atendimento/migration-job.yaml` e `k8s/estoque/migration-job.yaml`
4. Configurar todos os secrets no repositório GitHub
5. Validar pipeline em PR (apenas CI deve rodar)
6. Fazer push em `main` e validar pipeline completa (CI + CD)
7. Verificar rollout no cluster com `kubectl get pods -n oficina-mecanica`
