# Diagrama de Sequência — Autenticação via CPF

Fluxo completo desde a submissão do CPF até o consumo de uma rota protegida do Atendimento, conforme [RFC-003](../rfcs/RFC-003-estrategia-de-autenticacao.md).

```mermaid
sequenceDiagram
    actor Cliente
    participant APIGW as API Gateway
    participant Lambda as Lambda (.NET)
    participant RDS as RDS PostgreSQL
    participant SM as Secrets Manager
    participant Atend as Atendimento (EKS)

    Cliente->>APIGW: POST /auth/cpf { cpf }
    APIGW->>Lambda: invoca

    Lambda->>Lambda: valida formato do CPF

    alt CPF com formato inválido
        Lambda-->>APIGW: 400 Bad Request
        APIGW-->>Cliente: 400 Bad Request
    else CPF com formato válido
        Lambda->>RDS: consulta cliente por CPF
        RDS-->>Lambda: cliente encontrado?

        alt cliente não encontrado
            Lambda-->>APIGW: 404 Not Found
            APIGW-->>Cliente: 404 Not Found
        else cliente encontrado, status inativo
            Lambda-->>APIGW: 403 Forbidden
            APIGW-->>Cliente: 403 Forbidden
        else cliente encontrado, status ativo
            Lambda->>SM: lê chave JWT compartilhada
            SM-->>Lambda: chave HMAC-SHA256
            Lambda->>Lambda: assina JWT (claims: cliente_id, exp)
            Lambda-->>APIGW: 200 { token }
            APIGW-->>Cliente: 200 { token }
        end
    end

    Note over Cliente,Atend: Cliente já possui o token — consumo de rota protegida

    Cliente->>APIGW: GET /ordens-servico\nAuthorization: Bearer {token}
    APIGW->>APIGW: valida assinatura e expiração do JWT

    alt token inválido ou expirado
        APIGW-->>Cliente: 401 Unauthorized
    else token válido
        APIGW->>Atend: proxy da requisição
        Atend->>RDS: consulta dados
        RDS-->>Atend: resultado
        Atend-->>APIGW: 200 OK
        APIGW-->>Cliente: 200 OK
    end
```

## Notas

- O JWT emitido pela Lambda usa a **mesma chave** já usada pelo Atendimento para validar tokens do fluxo interno (login de atendentes/mecânicos) — um único middleware de validação, independente de qual fluxo emitiu o token.
- A validação de assinatura/expiração do token nas rotas protegidas acontece no **API Gateway** (authorizer), antes de a requisição alcançar o pod do Atendimento — uma requisição com token inválido nunca chega ao cluster.
- Rotas públicas (ex: `GET /ordens-servico/acompanhar/{numero}`) não passam pelo authorizer — ver [CARD-30](../cards/05-fase3-aws/CARD-30-api-gateway-integracao.md).

Ver também [Diagrama de Componentes](diagrama-componentes.md).
