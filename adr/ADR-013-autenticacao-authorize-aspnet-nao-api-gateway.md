# ADR-013 — Autorização via [Authorize] no ASP.NET Core, não no API Gateway

**Status:** Aceito
**Data:** 2026-08-25
**Autores:** Time Tech Challenge — Fase 3

---

## Contexto

O [CARD-30](../cards/05-fase3-aws/CARD-30-api-gateway-integracao.md) previa originalmente um **JWT authorizer no API Gateway**: a validação do token emitido pela Lambda (CARD-29) aconteceria na borda, antes de a requisição alcançar o EKS, e a rota `GET /ordens-servico/acompanhar/{numero}` ficaria **pública** (sem exigir token).

Na implementação, dois problemas com esse desenho apareceram:

1. **Incompatibilidade de algoritmo**: o JWT authorizer nativo do API Gateway (`aws_apigatewayv2_authorizer` tipo `JWT`) exige RS256 com JWKS endpoint. O esquema de autenticação já usado no projeto (Atendimento valida login de atendentes/mecânicos) é **HMAC-SHA256** com chave simétrica compartilhada — trocar para RS256 exigiria reescrever a validação de token de todo o sistema, não só do novo fluxo de CPF.
2. **Rota de acompanhamento exposta**: manter `/acompanhar/{numero}` pública, como no card original, permite que qualquer pessoa consulte o andamento de qualquer ordem de serviço sabendo (ou adivinhando) o número — o requisito real do desafio é "proteger rotas sensíveis com autenticação via CPF", e essa é justamente a rota pensada para o cliente final autenticado, não uma rota de uso interno sem dono.

A alternativa de resolver (1) criando um **Lambda authorizer customizado** (uma segunda função Lambda só para validar o HMAC e devolver a policy IAM ao API Gateway) foi avaliada e descartada: o enunciado oficial do desafio exige "proteger rotas sensíveis com autenticação via CPF", sem prescrever em qual camada a validação ocorre. Adicionar uma segunda Lambda só para revalidar um token que o Atendimento já é capaz de validar seria uma camada redundante para o mesmo requisito.

---

## Decisão

A validação do JWT nas rotas sensíveis do Atendimento acontece via **`[Authorize(Roles = "Cliente")]` no próprio ASP.NET Core** (middleware de autenticação já existente, mesmo usado para o fluxo de login de atendentes/mecânicos), não no API Gateway.

- O API Gateway passa a fazer apenas **proxy genérico** (`ANY /{proxy+}` → ALB do EKS), sem authorizer anexado a rotas específicas.
- A rota `GET /atendimento/ordens-servico/acompanhar/{numero}` deixa de ser pública: passa a exigir `[Authorize(Roles = "Cliente")]`. Requisição sem token válido recebe **401 do ASP.NET Core** (não do API Gateway).
- O JWT emitido pela Lambda (CARD-29) usa a claim `role=Cliente`, reconhecida pelo mesmo middleware que já valida tokens de atendente/mecânico — um único ponto de validação de token no sistema, independente da origem (login interno ou Lambda de CPF).

Isso **substitui o critério de aceite do CARD-30** que previa authorizer no API Gateway e rota pública — o card é atualizado para refletir esta decisão.

---

## Alternativas consideradas

### Alternativa 1: JWT authorizer nativo do API Gateway (RS256 + JWKS)

Rejeitada — exigiria migrar todo o esquema de assinatura de token do projeto (login de atendente/mecânico incluído) de HMAC-SHA256 para RS256, fora do escopo do CARD-30/29.

### Alternativa 2: Lambda authorizer customizado (valida HMAC, devolve policy IAM)

Tecnicamente viável — mantém a validação na borda sem trocar o algoritmo. Rejeitada por ser uma segunda camada de validação redundante com a que o Atendimento já faz, sem exigência explícita do enunciado para a validação ocorrer no API Gateway. Simplicidade preferida sobre replicar a validação em dois lugares.

---

## Consequências

### Positivas
- Reaproveita o middleware de autenticação já existente e testado no Atendimento — nenhuma lógica de validação de token nova
- Um único esquema de assinatura (HMAC-SHA256) para todos os fluxos de autenticação do sistema
- Rota de acompanhamento de OS deixa de estar exposta sem autenticação, alinhado ao requisito de "proteger rotas sensíveis"

### Negativas e mitigações

| Risco | Mitigação |
|---|---|
| Requisições com token inválido chegam ao pod do Atendimento antes de serem rejeitadas (não são bloqueadas na borda) | Aceitável neste escopo — o Atendimento roda atrás do ALB dentro da VPC, sem exposição direta; o custo de processar um 401 no próprio serviço é desprezível |
| Diagrama de sequência e diagrama de componentes anteriores descreviam o fluxo antigo (authorizer no API Gateway, rota pública) | Corrigidos em conjunto com este ADR — ver [Diagrama de Sequência — Autenticação via CPF](../diagramas/diagrama-sequencia-autenticacao.md) |

---

## Referências

- [CARD-29 — Lambda de autenticação via CPF](../cards/05-fase3-aws/CARD-29-lambda-autenticacao.md)
- [CARD-30 — Integração API Gateway → EKS com autenticação JWT](../cards/05-fase3-aws/CARD-30-api-gateway-integracao.md)
- `Tech-challenge/src/Atendimento/OficinaMecanica.Atendimento.API/Adapters/In/Http/AcompanhamentoOSController.cs`
- `tech-challenge-infra-k8s/api-gateway.tf`
