# CARD-41 — CI/CD, gates de qualidade e deploy independente

**Tipo:** Plataforma / CI-CD
**Status:** To Do
**Depende de:** CARD-34, CARD-35, CARD-37, CARD-38, CARD-39
**Bloqueia:** CARD-42, CARD-43
**Repositórios:** Três repositórios de serviço e repositórios de infraestrutura necessários
**Decisão arquitetural:** ADR-014; ADR/RFC de stores CARD-35; convenções de CI/CD existentes em CARD-27

---

## Contexto

Fase 3 já possui pipelines independentes para os repositórios de aplicação/infra, mas a aplicação mantém Atendimento e Estoque no mesmo pipeline. Fase 4 exige pipeline independente para cada microsserviço, incluindo build, testes, qualidade e deploy automático em Kubernetes. Uma biblioteca/template compartilhado de workflow é aceitável somente se cada repositório puder validar e implantar seu serviço sem publicar os outros.

## Escopo

- Pipeline autônomo por OS, Operações e Billing; branch principal protegida, PR obrigatório e checks requeridos.
- Build e testes unitários/integração/BDD pertinentes ao repositório; análise SonarQube ou ferramenta equivalente.
- Gate de cobertura >=80% por serviço com relatório e falha explícita quando abaixo do limite.
- Build/push de imagem imutável e deploy somente do serviço alterado em Kubernetes.
- Dockerfile e manifests Kubernetes próprios por serviço, com pipeline de deploy correspondente. A plataforma do cluster pode ser compartilhada se a interpretação aprovada no CARD-34 permitir.
- Provisionamento de infraestrutura por ferramenta declarativa a escolher no CARD-34. Terraform pode ser reaproveitado da Fase 3, mas não é requisito explícito do enunciado da Fase 4.
- Segurança de supply chain e secrets: OIDC onde suportado; nenhuma credencial AWS/Mercado Pago em código/log.
- Smoke/health checks e estratégia de rollback de release compatível com banco e eventos.

## Critérios de aceite

- [ ] Existem três workflows verificáveis, um por serviço, e alteração em um serviço não publica/deploya os outros.
- [ ] Cada pipeline executa restore, build, testes, análise de qualidade e gate de cobertura próprio.
- [ ] O relatório mostra cobertura por serviço e mede o mesmo escopo (linhas/branches) declarado na política; limite mínimo >=80% falha o workflow se não atingido.
- [ ] Ao menos um teste BDD integrado requerido por CARD-40 roda em workflow adequado e evidencia dependências usadas.
- [ ] Imagens são tagueadas por commit/digest e deployment usa artefato imutável, não `latest`.
- [ ] Cada serviço tem Dockerfile e manifests Kubernetes versionados em seu repositório, com recursos suficientes para deploy e health check independentes.
- [ ] Merge aprovado implanta somente a unidade de serviço correspondente no cluster Kubernetes e aguarda rollout/health.
- [ ] Branch principal exige PR e checks de build/testes/qualidade, conforme proteção configurada.
- [ ] Se Terraform for escolhido para provisionamento, pipelines de plan/apply têm aprovação/permissões adequadas e state remoto/locking seguem ADR-011 quando aplicável; a ausência de Terraform não reprova o card por si só.
- [ ] Secrets vêm de secret store/ambiente seguro e são mascarados nos logs.
- [ ] Falha de rollout impede conclusão do deploy e há procedimento de rollback/redeploy documentado.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Entregar cada microsserviço de forma independente

  Cenário: Publicar alteração apenas no serviço Operações
    Dado que um pull request altera somente Operações
    Quando as checagens obrigatórias são executadas e o PR é integrado
    Então o pipeline valida e implanta Operações
    E imagens e deployments de OS e Billing não são alterados

  Cenário: Bloquear merge abaixo da cobertura mínima
    Dado que a cobertura do serviço está abaixo de 80 por cento
    Quando o workflow de CI termina
    Então o gate de cobertura falha
    E o check requerido impede o merge

  Cenário: Impedir deploy de imagem não validada
    Dado que build, testes ou análise de qualidade falharam
    Quando o pipeline avalia a etapa de deploy
    Então nenhum deployment é atualizado
    E o resultado da etapa que bloqueou é visível no PR
```

## Passos

1. Criar matriz repo → workflow → checks requeridos → ambiente/Deployment.
2. Implementar workflow para cada serviço; reutilizar ações/templates somente sem acoplamento de release.
3. Ativar cobertura e quality gate por serviço, verificando que o caminho inclui o projeto certo.
4. Configurar proteção de branch via GitHub e registrar prova do check obrigatório.
5. Automatizar imagem, deploy e verificação de rollout do serviço correspondente.
6. Definir e documentar ferramenta declarativa para provisionamento de infraestrutura; preferir Terraform por continuidade se aprovado no CARD-34.
7. Testar PR abaixo do gate e deploy de uma alteração em ambiente Kubernetes de demonstração.
8. Documentar provisionamento, acesso e desligamento dos ambientes usados.

## Evidências

- URLs dos três workflows, proteção de branch, cobertura por serviço, análise de qualidade e histórico de deploy.
- Prova de que alterar um serviço não produz release dos demais.
- Instrução de deploy/rollback e relatório dos manifests aplicados.