# CARD-32 — Documentação arquitetural e entrega final

**Tipo:** Documentação
**Status:** To Do
**Depende de:** CARD-26, CARD-27, CARD-28, CARD-29, CARD-30, CARD-31
**Bloqueia:** —
**Decisão arquitetural:** —

---

## Contexto

Consolidar os entregáveis exigidos pelo desafio: diagramas, RFCs, ADRs (já feitos ao longo dos cards anteriores), README por repositório, vídeo de demonstração e o PDF único de entrega no Portal do Aluno.

---

## Critérios de aceite

### Diagramas (em `tech-challenge-docs/diagramas/`)
- [x] Diagrama de Componentes: visão de nuvem, APIs, banco e monitoramento (atualizado para incluir API Gateway, Lambda, EKS, RDS, Datadog)
- [x] Diagrama de Sequência: fluxo de autenticação via CPF (cliente → API Gateway → Lambda → RDS → JWT → rota protegida via `[Authorize]`, ver ADR-013)
- [ ] Diagrama de Sequência: fluxo de abertura de ordem de serviço (já existente na Fase 2, revisar se algo mudou)

### RFCs (em `tech-challenge-docs/rfcs/`)
- [x] RFC-001: escolha da nuvem (AWS vs Azure vs GCP)
- [x] RFC-002: escolha do banco de dados gerenciado (RDS PostgreSQL)
- [x] RFC-003: estratégia de autenticação (CPF + JWT via Lambda) — revisar se reflete a decisão final do ADR-013 (validação no Atendimento, não no API Gateway)

### ADRs
- [x] ADR-009 (migração AWS + repos) já registrado
- [x] ADR-012 (observabilidade Datadog, CARD-31) já registrado
- [x] ADR-013 (autorização via `[Authorize]` no Atendimento, não authorizer no API Gateway) já registrado
- [ ] Justificativa formal da escolha do banco de dados com diagrama ER e explicação dos relacionamentos (pode reaproveitar/atualizar ADR-007 e ADR-004)

### READMEs
- [ ] `Tech-challenge/README.md`: propósito, tecnologias, execução/deploy, diagrama específico, link Swagger/Postman
- [ ] `tech-challenge-lambda/README.md`: idem, com payload de entrada/saída da função
- [ ] `tech-challenge-infra-k8s/README.md`: idem, com outputs do Terraform
- [ ] `tech-challenge-infra-db/README.md`: idem, com outputs do Terraform
- [ ] `tech-challenge-docs/README.md`: índice de ADRs, RFCs, diagramas e cards

### Vídeo de demonstração (até 15 min, YouTube/Vimeo)
- [ ] Autenticação com CPF
- [ ] Execução da pipeline CI/CD
- [ ] Deploy automatizado
- [ ] Consumo das APIs protegidas
- [ ] Dashboard de monitoramento com análise ao vivo
- [ ] Logs e traces em execução

### Entrega no Portal do Aluno
- [ ] PDF único com: links dos 5 repositórios (4 exigidos + docs), link do vídeo, links das documentações, confirmação de `soat-architecture` adicionado a todos os repositórios

---

## Passos

1. Atualizar diagrama de componentes com API Gateway, Lambda, EKS, RDS
2. Criar diagrama de sequência do fluxo de autenticação via CPF
3. Escrever as 3 RFCs
4. Revisar ADR-004/ADR-007 quanto ao banco (se o modelo mudou com RDS, atualizar)
5. Escrever/revisar README de cada um dos 5 repositórios
6. Gravar o vídeo de demonstração cobrindo os 6 itens exigidos, cronometrando para ficar ≤ 15 min
7. Publicar o vídeo (não listado, se preferir privacidade)
8. Montar o PDF único com todos os links e a confirmação do colaborador
9. Revisar o PDF contra o checklist do enunciado antes de submeter
