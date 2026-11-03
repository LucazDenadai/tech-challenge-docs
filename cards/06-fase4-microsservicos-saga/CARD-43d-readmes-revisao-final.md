# CARD-43d — READMEs dos serviços e revisão final da entrega

**Tipo:** Documentação / Entrega
**Status:** Em andamento — escopo definido (2026-11-03)
**Depende de:** CARD-40, CARD-41, CARD-42
**Bloqueia:** Submissão no portal
**Decisão arquitetural:** [ADR-021](../../adr/ADR-021-kubernetes-local-sem-aws-fase4.md) (Kind, sem Lambda), [ADR-022](../../adr/ADR-022-observabilidade-fase4-stack-otel-local.md) (observabilidade)
**Repositórios alvo:** `tech-challenge-os`, `tech-challenge-billing`, `tech-challenge-operacoes`, `tech-challenge-docs`

---

## Contexto

Os READMEs dos serviços precisam ser autocontidos (E-04, E-05): ownership, arquitetura, como testar e executar, Swagger, cobertura e deploy. A revisão final confere links, segredos e o upload no portal.

## Lacunas encontradas (2026-11-03)

Os três READMEs já explicam estrutura, configuração, execução local, API, mensageria e testes. Comparados com os entregáveis do enunciado, faltam:

| Entregável do enunciado | OS | Billing | Operações |
|---|---|---|---|
| Evidência de cobertura (prints ou links no README) | falta | falta | falta |
| Pipeline de CI/CD descrito | falta | falta | falta |
| Dockerfile e manifests Kubernetes: como implantar no Kind | falta | falta | falta |
| Escolha da Saga justificada no README | tabelas, sem o "por quê" | só link ao ADR-017 | só link ao ADR-017 |
| Swagger ou collection Postman | Swagger citado; Postman não | idem | idem |
| Monitoramento e rastreamento | só `Jaeger__Endpoint` | idem | idem |

**SonarCloud não roda.** Os segredos `SONAR_TOKEN` e `SONAR_ORGANIZATION` não existem nos três repositórios, então o passo do CI registra só um aviso (CARD-41a). O R-12 do enunciado ("SonarQube ou similar no CI") fica parcial até o dono da conta criar os projetos e os segredos. O README não deve afirmar que a análise roda.

## Decisões (2026-11-03)

- **Mesmo bloco nos três READMEs**, depois de "Testes", nesta ordem: "Saga: por que orquestrada", "CI/CD", "Cobertura", "Executar no Kubernetes (Kind)", "Observabilidade" e "Swagger e Postman". O leitor acha a mesma informação no mesmo lugar em qualquer serviço.
- **Badge do workflow `CI`** no topo de cada README, apontando para a aba Actions.
- **Cobertura com número e link:** tabela com linhas e branches do último CI verde da `main`, o commit e o link do run, onde fica o artefato `cobertura-<repositório>` (relatório HTML do ReportGenerator). Valores de 2026-10-14: OS 93,6% / 86,3%, Operações 97,1% / 84,4%, Billing 94,7% / 81,9%. O badge mostra o estado atual; a tabela fica como evidência datada.
- **Justificativa da Saga:** completa no OS, que é o orquestrador (o mesmo texto do PDF da entrega). Nos participantes, um parágrafo com o papel do serviço e o link ao README do OS e ao ADR-017.
- **SonarCloud descrito como está:** o passo existe e roda com os segredos; hoje o gate de cobertura é a verificação que bloqueia o merge.
- **Kind:** os comandos do `tech-challenge-infra-k8s` (`kind/subir.sh`) e a porta de cada serviço (OS 8090, Billing 8091, Operações 8092). Nada sobre AWS (ADR-021).

## Implementação (2026-11-03)

- **READMEs:** `tech-challenge-os` PR #12, `tech-challenge-billing` PR #13 e `tech-challenge-operacoes` PR #10. Cada um ganhou o badge do CI (no OS, também o do BDD) e as seções "Saga: por que orquestrada", "CI/CD", "Cobertura", "Executar no Kubernetes (Kind)", "Observabilidade" e "Swagger e Postman". O texto é o mesmo nos três; mudam o nome, a porta, o run e os números.
- **Matriz de requisitos:** linhas que ainda mostravam o baseline passaram a mostrar o estado atual (R-01 a R-05, R-07 a R-09, R-11, R-12, E-01, E-04, E-05, E-07, E-08). Os cards CARD-37, 38 e 39 deixaram de dizer "falta CI (CARD-41)".

### Revisão de segredos e dados pessoais

Busca nos cinco repositórios públicos (`git grep`) por tokens do Mercado Pago, JWT, tokens do GitHub e do Sonar, chaves AWS, chaves privadas, e-mails pessoais e CPF formatado.

| Achado | Decisão |
|---|---|
| Nenhum token, chave ou segredo | — |
| CPFs `111.444.777-35`, `123.456.789-09` e `529.982.247-25` em testes e no `postman/README.md` | São CPFs de exemplo, válidos só no dígito verificador. Ficam. |
| E-mail pessoal do autor no ADR-011 (alertas da Fase 3) | Mantido por decisão do autor (2026-11-03): a conta AWS não existe mais e o dado é dele. |

## Critérios de aceite

- [x] Cada README traz ownership dos dados, contratos, testes, execução local e no Kind, Swagger e link da cobertura no CI.
- [x] Revisão final sem segredos, CPF real ou dados pessoais publicados.
- [ ] Links do PDF e do vídeo abertos e conferidos antes do envio; submissão registrada.

## Passos

1. PR no `tech-challenge-os`: badge e seções novas, com a justificativa completa da Saga.
2. PR no `tech-challenge-billing` e no `tech-challenge-operacoes`: badge e seções novas.
3. Revisão final: segredos, CPF real e dados pessoais nos repositórios públicos; links do PDF e do README do `tech-challenge-docs`.
4. Atualizar a matriz de requisitos (E-04, E-05, R-11, R-12) e registrar a implementação aqui.
5. Depois do vídeo: link no PDF (CARD-43c), conferir os links e registrar a submissão.
