# CARD-43d — READMEs dos serviços e revisão final da entrega

**Tipo:** Documentação / Entrega
**Status:** To Do (2026-10-13)
**Depende de:** CARD-40, CARD-41, CARD-42
**Bloqueia:** Submissão no portal
**Decisão arquitetural:** [ADR-021](../../adr/ADR-021-kubernetes-local-sem-aws-fase4.md) (Kind, sem Lambda), [ADR-022](../../adr/ADR-022-observabilidade-fase4-stack-otel-local.md) (observabilidade)
**Repositórios alvo:** `tech-challenge-os`, `tech-challenge-billing`, `tech-challenge-operacoes`, `tech-challenge-docs`

---

## Contexto

Os READMEs dos serviços precisam ser autocontidos (E-04, E-05): ownership, arquitetura, como testar e executar, Swagger, cobertura e deploy. A revisão final confere links, segredos e o upload no portal.

## Critérios de aceite

- [ ] Cada README traz ownership dos dados, contratos, testes, execução local e no Kind, Swagger e link da cobertura no CI.
- [ ] Revisão final sem segredos, CPF real ou dados pessoais publicados.
- [ ] Links do PDF e do vídeo abertos e conferidos antes do envio; submissão registrada.
