# ADR-006 — Self-Hosted Runner para Deploy no CI/CD

**Status:** Superseded por [ADR-009](ADR-009-migracao-aws-e-separacao-repositorios.md)
**Data:** 2026-06-11  
**Autores:** Time Tech Challenge — Fase 2

---

## Contexto

O pipeline de CI/CD (CARD-14) exige que o job de deploy aplique manifestos Kubernetes num cluster local (Kind/Docker Desktop), conforme decisão do ADR-003. O runner padrão do GitHub Actions (`ubuntu-latest`) roda em servidores da nuvem da GitHub e não tem acesso à rede local onde o cluster está provisionado — tentativas de conexão resultam em `dial tcp 127.0.0.1: connect: connection refused`.

O requisito do Tech Challenge inclui explicitamente o deploy automatizado no cluster K8s como parte do pipeline de CD. A avaliação ocorre por meio do histórico de execuções visível no GitHub Actions e do vídeo demonstrativo gravado com o pipeline rodando.

---

## Decisão

Utilizar um **self-hosted runner** do GitHub Actions instalado na mesma máquina que hospeda o cluster Kubernetes local. Apenas o job `deploy` usa `runs-on: self-hosted` — os jobs `build-and-test` e `docker` continuam em `ubuntu-latest` para garantir ambiente limpo e reproduzível.

---

## Consequências

### Positivas
- O runner tem acesso direto ao `kubectl` e ao cluster local — o deploy funciona sem expor o cluster à internet
- Nenhum custo adicional de infraestrutura em nuvem
- O histórico de execuções fica visível no GitHub Actions para avaliação
- Jobs de build e teste continuam isolados no runner gerenciado do GitHub

### Negativas e mitigações

| Risco | Mitigação |
|---|---|
| O job `deploy` só executa enquanto o runner estiver ativo na máquina do desenvolvedor | Documentado no README — avaliação ocorre via histórico e vídeo, não por re-execução pelo professor |
| Ambiente do runner não é limpo entre execuções | Aceitável para CD; o `kubectl apply` é idempotente |

---

## Alternativas consideradas

### Alternativa 1: Kind dentro do runner `ubuntu-latest`

Criar um cluster Kind efêmero no próprio runner do GitHub a cada execução de pipeline.

**Prós:** Totalmente reproduzível por qualquer pessoa, sem dependência de máquina local.  
**Contras:** Adiciona 3-5 minutos ao pipeline para criar o cluster; imagens do GHCR precisariam ser carregadas via `kind load`; complexidade operacional maior.

**Por que não escolhemos agora:** o self-hosted resolve o problema imediato com menos risco de quebra; Kind pode ser adotado em iteração futura se necessário.

### Alternativa 2: Cluster em nuvem (GKE, EKS, AKS)

Provisionar um cluster gerenciado na nuvem acessível pelo runner do GitHub.

**Prós:** Elimina dependência de máquina local; totalmente reproduzível.  
**Contras:** Custo financeiro; fora do escopo do desafio que aceita cluster local.

**Por que não escolhemos:** o ADR-003 já decidiu por cluster local para o desafio.

---

## Referências

- [GitHub Actions — Self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners)
- [ADR-003 — Arquitetura de Deploy no Kubernetes](ADR-003-arquitetura-kubernetes.md)
- [CARD-14 — Pipeline CI/CD](../cards/04-cicd/CARD-14-pipeline-cicd.md)
