# Relatório de Avaliação — Tech Challenge Fase 2

**Data da avaliação:** 2026-06-11  
**Branch avaliada:** `feature/cicd`  
**Avaliador:** Claude (revisão pré-entrega)

---

## Status geral

Projeto tecnicamente sólido, bem estruturado e com nível de maturidade acima da média para um trabalho de pós-graduação. A entrega cobre todos os requisitos obrigatórios do enunciado, com pontos a corrigir antes da entrega final.

---

## Itens descartados (não serão alterados)

| Item | Decisão |
|---|---|
| **Q2 — 11 endpoints no OrdensServicoController** | Mantido por conveniência. Não há problema arquitetural impeditivo. |

---

## Itens a implementar

### CARD-17 — ADR: banco compartilhado com schemas separados

**O que é:** Falta uma ADR documentando a decisão de usar um único PostgreSQL com dois schemas (`atendimento` e `estoque`) em vez de databases separados por serviço.

**Por que importa:** Viola o princípio de isolamento de dados de microserviços. Um avaliador vai questionar. A ADR justifica a decisão de forma consciente.

**Decisão já tomada:** Um PostgreSQL, dois schemas. Motivo: simplicidade de operação para ambiente acadêmico — Terraform, K8s e CI/CD já estão calibrados para esse modelo. O custo de migrar para dois databases superaria o benefício no contexto do projeto.

**Arquivo a criar:** `docs/adr/ADR-007-banco-compartilhado-schemas-separados.md`

**Esforço:** Baixo — só documentação.

---

### CARD-18 — Email: implementar atualização de status via disparo de email

**O que é:** O enunciado exige "Atualização de status da OS via alguma ferramenta como email." A interpretação adotada é: ao atualizar o status da OS, um email é disparado notificando o cliente. O `EmailStub` já existe; o adapter real precisa ser validado e o fluxo demonstrado.

**Situação atual:**
- `EmailStub` existe em `Infrastructure/Adapters/Out/Stubs/`
- Adapter de email real existe em `Infrastructure/Adapters/Out/Email/`
- Não está claro se o disparo acontece de fato no use case `AtualizarStatusOSUseCase`

**O que precisa ser feito:**
1. Confirmar que `AtualizarStatusOSUseCase` chama o port de email após mudança de status
2. Confirmar que o adapter real está registrado no DI (não o stub) em produção
3. Confirmar que as variáveis SMTP estão nos secrets do K8s e no docker-compose
4. Demonstrar no vídeo: atualizar status → email recebido na caixa de entrada

**Esforço:** Médio — pode ser que já funcione e precise só de validação e demonstração.

---

### CARD-19 — CI/CD: adicionar `needs: build-and-test` no job docker

**O que é:** O job `docker` não depende de `build-and-test`. Em um push direto para `main` sem PR, a imagem Docker é buildada e publicada sem os testes terem rodado.

**Situação atual no `.github/workflows/dotnet.yml`:**
```yaml
docker:
  name: Build & Push Docker Image
  runs-on: ubuntu-latest
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  # sem needs: build-and-test
```

**O que precisa ser feito:**
1. Adicionar `needs: build-and-test` no job `docker`
2. Ajustar o `if` do `build-and-test` para rodar também em push para main (hoje só roda em PR):
   ```yaml
   build-and-test:
     if: true  # remover o filtro de PR, deixar rodar sempre
   ```
   Ou manter o filtro e aceitar o risco documentado.

**Decisão a tomar:** rodar testes em todo push para main tem custo de tempo de pipeline (~3-5 min a mais). Para o contexto acadêmico, vale a pena para garantir que o vídeo mostre um pipeline sem esse gap.

**Esforço:** Baixo — mudança de 2-3 linhas no YAML.

---

### CARD-20 — CI/CD: orquestrar migration-job no pipeline

**O que é:** O `migration-job.yaml` existe e é aplicado via `kubectl apply -f k8s/`, mas o pipeline não aguarda o Job completar antes de subir o Deployment. Não há garantia de que as migrations rodam antes da aplicação tentar acessar o banco.

**Situação atual:**
```yaml
- name: Aplicar manifestos K8s
  run: kubectl apply -f k8s/ -n oficina-mecanica
  # aplica tudo junto — job e deployment sem ordenação
```

**O que precisa ser feito:**
1. Separar o apply do migration-job do apply dos deployments
2. Adicionar `kubectl wait` para aguardar o Job concluir antes de aplicar o Deployment:
   ```powershell
   kubectl apply -f k8s/atendimento/migration-job.yaml -n oficina-mecanica
   kubectl wait --for=condition=complete job/atendimento-migration --timeout=120s -n oficina-mecanica
   kubectl apply -f k8s/atendimento/deployment.yaml -n oficina-mecanica
   ```
3. Repetir para o serviço de estoque

**Esforço:** Baixo-médio — requer reestruturar o step de apply no pipeline.

---

### CARD-21 — Terraform: remover tfstate e tfvars do repositório

**O que é:** `terraform.tfstate` e `terraform.tfvars` estão commitados no repositório. O tfstate pode conter dados sensíveis; o tfvars contém a senha do banco em texto claro.

**Situação atual:**
- `infra/terraform.tfstate` — no repositório
- `infra/terraform.tfstate.backup` — no repositório
- `infra/terraform.tfvars` — provavelmente no repositório (tem o `terraform.tfvars.example`)

**O que precisa ser feito:**
1. Verificar o `.gitignore` — adicionar:
   ```
   infra/terraform.tfstate
   infra/terraform.tfstate.backup
   infra/terraform.tfvars
   ```
2. Remover os arquivos do tracking do git:
   ```
   git rm --cached infra/terraform.tfstate infra/terraform.tfstate.backup
   git rm --cached infra/terraform.tfvars
   ```
3. Documentar no `infra/README.md` que o tfvars deve ser criado localmente a partir do `.example`

**Esforço:** Baixo — mas tem impacto em git history. Fazer com cuidado.

---

### CARD-22 — Terraform: documentar limitação do local-exec Windows-only

**O que é:** O módulo `database` usa `local-exec` com PowerShell para criar os schemas. Isso funciona apenas em Windows. Se alguém tentar rodar `terraform apply` em Linux/Mac, vai quebrar.

**Situação atual no `infra/modules/database/main.tf`:**
```hcl
provisioner "local-exec" {
  interpreter = ["PowerShell", "-Command"]
  command     = <<-EOT
    ...PowerShell script...
  EOT
}
```

**O que precisa ser feito:**
- Opção A (recomendada para entrega): documentar explicitamente no `infra/README.md` que o Terraform só funciona em Windows, e que é um trade-off intencional para o ambiente de desenvolvimento local
- Opção B (ideal em produção): reescrever o provisioner usando `bash` ou substituir por um recurso nativo (ex: usar um `null_resource` com script cross-platform)

Para o contexto acadêmico, **a opção A basta** — o avaliador aceita desde que documentado.

**Esforço:** Muito baixo — só documentação.

---

### CARD-23 — Testes: corrigir coverage no SonarCloud e limpar README

**O que é:** O SonarCloud não está capturando a cobertura de código. O README menciona "107 testes" e implica cobertura, mas sem evidência real.

**Situação atual:**
- O pipeline coleta arquivos `coverage.opencover.xml`
- O SonarCloud recebe o path `**/coverage.opencover.xml`
- Mas o SonarCloud não exibe a cobertura — possivelmente problema de caminho ou formato

**O que precisa ser feito:**
1. Investigar por que o coverage não aparece no SonarCloud:
   - Verificar se o arquivo `coverage.opencover.xml` é gerado na pasta esperada
   - Confirmar que o `coverlet.runsettings` está correto
   - Confirmar que o `sonar.cs.opencover.reportsPaths` aponta para o lugar certo
2. Se não for possível corrigir antes da entrega: remover do README qualquer menção a cobertura que não pode ser evidenciada
3. Corrigir o nome do teste `ExecutarAsync_DadosValidos_CriaOSComStatusRecebida` para refletir o que realmente verifica, ou adicionar a assertion do status:
   ```csharp
   // adicionar no teste existente:
   _osRepoMock.Verify(r => r.AdicionarAsync(
       It.Is<DomainOS>(os => os.Status == StatusOrdemServico.Recebida), default),
       Times.Once);
   ```

**Esforço:** Médio — investigação de configuração do SonarCloud + pequena correção de teste.

---

### CARD-24 — K8s: documentar image hardcoded no deployment.yaml

**O que é:** O `deployment.yaml` do Atendimento referencia `image: ghcr.io/lucazdenadai/tech-challenge/oficina-atendimento:latest` com o nome do usuário hardcoded. O pipeline atualiza a imagem via `kubectl set image`, então na prática o YAML inicial é sobrescrito — mas é um ponto de fragilidade.

**O que precisa ser feito:**
- Opção A: deixar como está e documentar que o manifest YAML é apenas o estado inicial e o pipeline sempre faz `kubectl set image` como fonte de verdade
- Opção B: usar um placeholder (`image: ATENDIMENTO_IMAGE`) e substituir no pipeline com `sed` antes do apply

Para a entrega, **a opção A é suficiente** — acrescentar um comentário no próprio YAML e no README explicando o fluxo.

**Esforço:** Muito baixo.

---

## Resumo de prioridade

| Card | Título | Prioridade | Esforço |
|---|---|---|---|
| **CARD-17** | ADR banco compartilhado | Alta | Baixo |
| **CARD-19** | CI/CD: needs build-and-test | Alta | Baixo |
| **CARD-21** | Terraform: remover tfstate/tfvars do repo | Alta | Baixo |
| **CARD-18** | Email: validar e demonstrar disparo | Alta | Médio |
| **CARD-20** | CI/CD: orquestrar migration antes do deploy | Média | Médio |
| **CARD-23** | Testes: coverage SonarCloud + corrigir teste | Média | Médio |
| **CARD-22** | Terraform: documentar Windows-only | Baixa | Muito baixo |
| **CARD-24** | K8s: documentar image hardcoded | Baixa | Muito baixo |

---

## Itens que já estão bem (não alterar)

- Arquitetura Hexagonal bem aplicada em ambos os serviços
- Máquina de estados do domínio `OrdemServico` com transições protegidas
- Comunicação assíncrona via RabbitMQ justificada em ADR
- HPA configurado com CPU e memória
- Init containers aguardando Postgres
- Stack de observabilidade completa (Prometheus, Grafana, Loki, Jaeger)
- Pipeline com pinagem de actions por hash de commit
- Testes de integração com Testcontainers (banco real)
- README completo com arquitetura, instruções e Postman collection
- 6 ADRs documentando as principais decisões
