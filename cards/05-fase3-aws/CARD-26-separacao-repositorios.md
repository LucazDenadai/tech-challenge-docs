# CARD-26 — Separação em 5 repositórios

**Tipo:** Infra
**Status:** To Do
**Depende de:** —
**Bloqueia:** CARD-27, CARD-28, CARD-29, CARD-30
**Decisão arquitetural:** [ADR-009](../../adr/ADR-009-migracao-aws-e-separacao-repositorios.md)

---

## Contexto

A Fase 3 exige 4 repositórios separados (Lambda, Infra Kubernetes, Infra Banco, Aplicação principal), cada um com CI/CD e deploy automático. O ADR-009 acrescenta um 5º repositório (`tech-challenge-docs`) para centralizar ADRs, RFCs, diagramas e os **cards de execução** (`docs/cards/`), evitando duplicação entre os 4 repositórios exigidos e mantendo um único histórico de planejamento entre app, infra e lambda.

Este card cria os repositórios vazios no GitHub e move o conteúdo já existente deste monorepo (`infra/`, `docs/adr/`, `docs/cards/`) para seus destinos, deixando este repositório (`Tech-challenge`) apenas com `src/`, `tests/`, `k8s/` (manifests de aplicação) e seu próprio CI/CD.

---

## Critérios de aceite

- [ ] 5 repositórios criados no GitHub: `tech-challenge-docs`, `tech-challenge-lambda`, `tech-challenge-infra-k8s`, `tech-challenge-infra-db`, `Tech-challenge`
- [ ] Branch `main` protegida em todos (sem commit direto, PR obrigatório) — CARD-27 cuida da configuração de CI/CD, aqui só a proteção de branch
- [ ] Usuário `soat-architecture` adicionado como colaborador em todos os 5
- [ ] `tech-challenge-docs` contém todos os ADRs (001 a 009), RFCs, diagramas e todo o conteúdo de `docs/cards/` (incluindo este card)
- [ ] `infra/modules/cluster` migrado para `tech-challenge-infra-k8s` (a ser reescrito em CARD-28 para AWS)
- [ ] `infra/modules/database` migrado para `tech-challenge-infra-db` (a ser reescrito em CARD-28 para AWS)
- [ ] `tech-challenge-lambda` criado vazio com estrutura mínima de solution .NET
- [ ] Este repositório (`Tech-challenge`) mantém `src/`, `tests/`, `k8s/atendimento`, `k8s/estoque`, `k8s/postgres`, `k8s/rabbitmq`, `k8s/observabilidade`, `k8s/namespace.yaml`
- [ ] READMEs de todos os 5 repositórios linkam entre si (app/lambda/infra-k8s/infra-db → docs; docs → os outros 4)
- [ ] `git log` histórico não precisa ser preservado na migração (cópia de arquivos, não `git filter-repo`) — simplicidade sobre rastreabilidade de histórico antigo

---

## Estrutura final por repositório

```
tech-challenge-docs/
├── adr/              ← ADR-001 a ADR-009
├── cards/            ← todo o conteúdo atual de docs/cards/
├── rfcs/
├── diagramas/
└── README.md

tech-challenge-lambda/
├── src/
├── tests/
└── README.md

tech-challenge-infra-k8s/
├── main.tf, variables.tf, outputs.tf, versions.tf
├── modules/cluster/   ← reescrito para EKS em CARD-28
└── README.md

tech-challenge-infra-db/
├── main.tf, variables.tf, outputs.tf, versions.tf
├── modules/database/  ← reescrito para RDS em CARD-28
└── README.md

Tech-challenge/ (este repositório, renomeado se necessário)
├── src/Atendimento/, src/Estoque/
├── tests/
├── k8s/
└── README.md
```

---

## Passos

1. Criar os 5 repositórios vazios no GitHub via `gh repo create`
2. Adicionar `soat-architecture` como colaborador em todos
3. Copiar `docs/adr/`, `docs/cards/` (todo o histórico, incluindo Fase 2), RFCs e diagramas para `tech-challenge-docs`
4. Copiar `infra/modules/cluster` para `tech-challenge-infra-k8s` (estrutura de pastas, sem reescrever ainda)
5. Copiar `infra/modules/database` para `tech-challenge-infra-db` (estrutura de pastas, sem reescrever ainda)
6. Criar solution .NET vazia em `tech-challenge-lambda`
7. Remover `infra/` e `docs/adr/`, `docs/cards/` deste repositório após confirmar a cópia
8. Atualizar README deste repositório removendo seções de Terraform/infra que migraram, linkando para `tech-challenge-docs`
9. Configurar proteção de branch `main` nos 5 repositórios (sem CI/CD ainda — isso é CARD-27)
