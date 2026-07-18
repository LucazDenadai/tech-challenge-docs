# CARD-25 — CI/CD: adicionar step de Terraform no pipeline

**Tipo:** Infra  
**Status:** To Do  
**Depende de:** CARD-13, CARD-14  
**Bloqueia:** nenhum

---

## Contexto

O enunciado da Fase 2 exige que o pipeline CI/CD execute o provisionamento da infraestrutura com Terraform, além do build, testes, Docker e deploy K8s. Atualmente o Terraform é executado manualmente pelo desenvolvedor antes de rodar o pipeline. Essa separação não atende o critério explícito do edital:

> "Pipeline de CI/CD configurada (...) que execute: (...) Deploy do banco de dados. Aplicação dos manifestos YAML no cluster."

O step de Terraform cobre o provisionamento do cluster Kind e do banco de dados — o que completa o requisito de IaC integrado ao CI/CD.

---

## Critérios de aceite

- [ ] Pipeline executa `terraform init` e `terraform apply` antes do deploy K8s
- [ ] Step só roda em push para `main` (não em PRs)
- [ ] Step roda no runner `self-hosted` (acesso à rede local onde o Kind será criado)
- [ ] `terraform apply` é idempotente — não recria o cluster se já existir
- [ ] Senha do banco vem de GitHub Secret, não de `terraform.tfvars`
- [ ] README documenta que o Terraform é orquestrado pelo pipeline

---

## Estrutura do job no dotnet.yml

Adicionar job `infra` entre `build-and-test` e `docker`:

```yaml
infra:
  name: Provisionar Infraestrutura (Terraform)
  runs-on: self-hosted
  needs: build-and-test
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'

  steps:
    - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@b9cd54531c595178fb73c78e1df1e5e2e994d10c # v3.1.2
      with:
        terraform_version: "~> 1.6"

    - name: Terraform Init
      working-directory: infra
      shell: powershell
      run: terraform init

    - name: Terraform Apply
      working-directory: infra
      shell: powershell
      env:
        TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}
      run: terraform apply -auto-approve -var="db_password=$env:TF_VAR_db_password"
```

O job `docker` passa a depender de `infra`:

```yaml
docker:
  needs: [build-and-test, infra]
```

E o job `deploy` mantém `needs: docker`.

---

## Ordem final da pipeline

```
build-and-test (ubuntu-latest)
       ↓
    infra (self-hosted) ← novo
       ↓
    docker (ubuntu-latest)
       ↓
    deploy (self-hosted)
```

---

## Observação sobre idempotência

O provider `tehcyx/kind` verifica se o cluster já existe antes de criar. O `docker_container` do postgres também é idempotente. Se o cluster já estiver provisionado (run anterior), o `terraform apply` completa sem recriar nada — apenas confirma o estado desejado.

---

## Passos

1. Adicionar job `infra` no `.github/workflows/dotnet.yml`
2. Adicionar `needs: [build-and-test, infra]` no job `docker`
3. Adicionar secret `DB_PASSWORD` no repositório GitHub (se ainda não existir)
4. Testar push em `main` e verificar que o cluster é provisionado antes do deploy
5. Atualizar README com o fluxo completo do pipeline incluindo o step de Terraform
