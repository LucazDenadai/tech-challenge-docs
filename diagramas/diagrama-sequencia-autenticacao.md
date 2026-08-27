# Diagrama de Sequência — Autenticação via CPF

Fluxo completo desde a submissão do CPF até o consumo de uma rota protegida do Atendimento, conforme [RFC-003](../rfcs/RFC-003-estrategia-de-autenticacao.md) e [ADR-013](../adr/ADR-013-autenticacao-authorize-aspnet-nao-api-gateway.md).

```mermaid
sequenceDiagram
    actor Cliente
    participant APIGW as API Gateway
    participant Lambda as Lambda (.NET)
    participant RDS as RDS PostgreSQL
    participant Atend as Atendimento (EKS)

    Cliente->>APIGW: POST /atendimento/auth/cpf { cpf }
    APIGW->>Lambda: invoca (integração AWS_PROXY)

    Lambda->>Lambda: valida formato do CPF

    alt CPF com formato inválido
        Lambda-->>APIGW: 400 Bad Request
        APIGW-->>Cliente: 400 Bad Request
    else CPF com formato válido
        Lambda->>RDS: consulta cliente por CPF (schema atendimento)
        RDS-->>Lambda: cliente encontrado?

        alt cliente não encontrado
            Lambda-->>APIGW: 404 Not Found
            APIGW-->>Cliente: 404 Not Found
        else cliente encontrado, status inativo
            Lambda-->>APIGW: 403 Forbidden
            APIGW-->>Cliente: 403 Forbidden
        else cliente encontrado, status ativo
            Lambda->>Lambda: assina JWT HMAC-SHA256\n(claims: cliente_id, role=Cliente, exp)
            Lambda-->>APIGW: 200 { token }
            APIGW-->>Cliente: 200 { token }
        end
    end

    Note over Cliente,Atend: Cliente já possui o token — consumo de rota protegida

    Cliente->>APIGW: GET /atendimento/ordens-servico/acompanhar/{numero}\nAuthorization: Bearer {token}
    APIGW->>Atend: proxy genérico (sem authorizer na borda — ADR-013)
    Atend->>Atend: [Authorize(Roles="Cliente")]\nvalida assinatura e expiração do JWT

    alt token ausente, inválido ou expirado
        Atend-->>APIGW: 401 Unauthorized
        APIGW-->>Cliente: 401 Unauthorized
    else token válido
        Atend->>RDS: consulta dados da ordem de serviço
        RDS-->>Atend: resultado
        Atend-->>APIGW: 200 OK
        APIGW-->>Cliente: 200 OK
    end
```

## Notas

- O JWT emitido pela Lambda usa a **mesma chave HMAC-SHA256** já usada pelo Atendimento para validar tokens do fluxo interno (login de atendentes/mecânicos) — um único middleware de validação (`[Authorize]` do ASP.NET Core), independente de qual fluxo emitiu o token.
- A validação de assinatura/expiração do token acontece **dentro do Atendimento** (`[Authorize(Roles="Cliente")]`), não no API Gateway — decisão registrada em [ADR-013](../adr/ADR-013-autenticacao-authorize-aspnet-nao-api-gateway.md). O API Gateway faz apenas proxy genérico (`ANY /{proxy+}`) para o ALB do EKS.
- A rota `GET /ordens-servico/acompanhar/{numero}` **exige** JWT válido com role `Cliente` — não é mais uma rota pública (mudança em relação ao desenho original do CARD-30).

Ver também [Diagrama de Componentes](diagrama-componentes.md).
