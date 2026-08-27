# CARD-30 — Integração API Gateway → EKS com autenticação JWT

**Tipo:** Infra/Segurança
**Status:** Concluído
**Depende de:** CARD-28, CARD-29
**Bloqueia:** CARD-31
**Decisão arquitetural:** [ADR-009](../../adr/ADR-009-migracao-aws-e-separacao-repositorios.md), [ADR-013](../../adr/ADR-013-autenticacao-authorize-aspnet-nao-api-gateway.md)

---

## Contexto

Conectar as três pontas: API Gateway recebe a requisição, rotas sensíveis exigem o JWT emitido pela Lambda (CARD-29), requisições são roteadas para o Atendimento rodando no EKS (CARD-28).

**Mudança em relação ao desenho original** (registrada em [ADR-013](../../adr/ADR-013-autenticacao-authorize-aspnet-nao-api-gateway.md)): a validação do JWT não acontece mais no API Gateway (JWT authorizer nativo exigiria RS256, incompatível com o HMAC-SHA256 já usado no projeto). O API Gateway faz apenas proxy genérico; a validação acontece no próprio Atendimento via `[Authorize(Roles="Cliente")]`. A rota `GET /ordens-servico/acompanhar/{numero}` deixou de ser pública — agora exige JWT, por ser a rota pensada para o cliente final.

---

## Critérios de aceite

- [x] API Gateway tem rota `POST /atendimento/auth/cpf` apontando para a Lambda (CARD-29)
- [x] Rota `GET /atendimento/ordens-servico/acompanhar/{numero}` exige JWT válido com role `Cliente` — validado via `[Authorize(Roles="Cliente")]` no Atendimento (não no API Gateway — ver ADR-013)
- [x] Requisição sem token ou com token inválido/expirado retorna 401 (emitido pelo ASP.NET Core)
- [x] Requisição com token válido chega ao Atendimento no EKS e é processada normalmente
- [x] Teste end-to-end documentado: login via CPF → token → chamada a rota protegida → 200 (`Acompanhar_ComTokenDeCliente_DeveRetornar200`)
- [x] Teste end-to-end documentado: chamada a rota protegida sem token → 401 (`Acompanhar_SemToken_DeveRetornar401`)

---

## Passos

1. ~~Configurar JWT authorizer no API Gateway~~ — descartado, ver ADR-013
2. Configurar rota `POST /atendimento/auth/cpf` → integração Lambda (AWS_PROXY)
3. Configurar rota proxy genérica (`ANY /{proxy+}`) → ALB do EKS
4. Trocar `[AllowAnonymous]` por `[Authorize(Roles="Cliente")]` em `AcompanhamentoOSController`
5. Testar o fluxo completo: CPF → JWT → chamada autenticada
6. Testar rejeição: chamada sem token
7. Documentar o fluxo no [Diagrama de Sequência — Autenticação via CPF](../../diagramas/diagrama-sequencia-autenticacao.md)
