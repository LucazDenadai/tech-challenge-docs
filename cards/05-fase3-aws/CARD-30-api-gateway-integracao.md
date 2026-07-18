# CARD-30 — Integração API Gateway → EKS com autenticação JWT

**Tipo:** Infra/Segurança
**Status:** To Do
**Depende de:** CARD-28, CARD-29
**Bloqueia:** CARD-31
**Decisão arquitetural:** [ADR-009](../../adr/ADR-009-migracao-aws-e-separacao-repositorios.md)

---

## Contexto

Conectar as três pontas: API Gateway recebe a requisição, rotas sensíveis exigem o JWT emitido pela Lambda (CARD-29), rotas autenticadas são roteadas para o Atendimento rodando no EKS (CARD-28). Rotas públicas (ex: `GET /ordens-servico/acompanhar/{numero}`, já sem autenticação na Fase 2) continuam sem exigir JWT.

---

## Critérios de aceite

- [ ] API Gateway tem rota `POST /auth/cpf` apontando para a Lambda (CARD-29)
- [ ] Rotas sensíveis do Atendimento (`/clientes`, `/veiculos`, `/servicos`, `/ordens-servico` exceto `/acompanhar/{numero}`) exigem JWT válido — autorizador do API Gateway (Lambda authorizer ou JWT authorizer nativo) valida o token antes de rotear ao EKS
- [ ] Rota pública `/ordens-servico/acompanhar/{numero}` não exige JWT
- [ ] Requisição sem token ou com token inválido/expirado retorna 401 no nível do API Gateway (nunca chega ao pod)
- [ ] Requisição com token válido chega ao Atendimento no EKS e é processada normalmente
- [ ] Teste end-to-end documentado: login via CPF → token → chamada a rota protegida → 200
- [ ] Teste end-to-end documentado: chamada a rota protegida sem token → 401

---

## Passos

1. Configurar JWT authorizer no API Gateway (validação de assinatura, `iss`, `exp` — mesma chave usada pela Lambda e pelo Atendimento)
2. Configurar rota `POST /auth/cpf` → integração Lambda
3. Configurar rotas proxy → ALB do EKS, com o authorizer anexado às rotas sensíveis
4. Confirmar que a rota pública de acompanhamento não tem authorizer anexado
5. Testar o fluxo completo: CPF → JWT → chamada autenticada
6. Testar rejeição: chamada sem token, chamada com token expirado
7. Documentar o fluxo no README de `tech-challenge-infra-k8s` com diagrama de sequência (ligar ao diagrama exigido na documentação final)
