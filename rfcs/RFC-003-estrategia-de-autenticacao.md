# RFC-003 — Estratégia de autenticação (CPF + JWT via Lambda)

**Status:** Aceito
**Data:** 2026-07-18
**Autores:** Time Tech Challenge — Fase 3
**ADR relacionado:** [ADR-009](../adr/ADR-009-migracao-aws-e-separacao-repositorios.md), [ADR-013](../adr/ADR-013-autenticacao-authorize-aspnet-nao-api-gateway.md)
**Card relacionado:** [CARD-29](../cards/05-fase3-aws/CARD-29-lambda-autenticacao.md), [CARD-30](../cards/05-fase3-aws/CARD-30-api-gateway-integracao.md)

---

## Contexto

O desafio exige proteger rotas sensíveis da aplicação com autenticação via CPF, delegada a uma Function Serverless que:

1. Valida o formato do CPF;
2. Consulta a existência e o status do cliente na base de dados;
3. Gera e devolve um token JWT válido para consumo das APIs protegidas.

A Fase 2 já usa JWT (HMAC-SHA256) com login por e-mail/senha para usuários internos (atendentes, mecânicos, admin) — ver seção "Autenticação" do README de `Tech-challenge`. A Fase 3 introduz um **segundo fluxo de autenticação**, para clientes externos, baseado em CPF, sem senha — o cliente já está cadastrado no sistema (pelo atendente) e usa o CPF como identificador para obter acesso.

---

## Proposta

Implementar a autenticação por CPF como uma AWS Lambda (.NET) exposta via rota do API Gateway, seguindo o fluxo:

```
Cliente → POST /auth/cpf {cpf} → API Gateway → Lambda
                                                  │
                                                  ├─ 1. Valida formato do CPF (CpfValidator)
                                                  ├─ 2. Consulta cliente no RDS (schema atendimento)
                                                  ├─ 3. Verifica status do cliente (ativo/inativo)
                                                  └─ 4. Assina JWT (mesma chave HMAC-SHA256 do Atendimento)
                                                          │
                                                          ▼
                                              200 { token } | 400 | 404 | 403
```

O JWT emitido pela Lambda é validado pelo **mesmo mecanismo** já usado pelas rotas protegidas do Atendimento (mesma chave secreta, injetada como variável de ambiente/Kubernetes Secret via GitHub Secrets em ambos os repositórios) — não é um sistema de autenticação paralelo, é uma segunda *forma de obter* um token com a mesma validade e formato. A validação em si acontece dentro do Atendimento via `[Authorize]`, não no API Gateway — ver [ADR-013](../adr/ADR-013-autenticacao-authorize-aspnet-nao-api-gateway.md).

### Por que Lambda (Function Serverless) e não um endpoint dentro do Atendimento

O desafio pede explicitamente que essa validação seja uma Function Serverless, não um endpoint da API principal. Isso também faz sentido tecnicamente: a autenticação por CPF é um fluxo de borda (entrada no sistema), de baixo volume e sem estado, um caso de uso natural para serverless — escala a zero quando não há logins acontecendo, sem manter um pod do Atendimento reservado só para isso.

### Por que .NET na Lambda (não Node.js)

Ver [ADR-009](../adr/ADR-009-migracao-aws-e-separacao-repositorios.md), seção "Lambda: .NET/C#, não Node.js": mantém a mesma stack do restante do projeto. Cold start (~200-400ms) é aceitável para um fluxo de login, chamado uma vez por sessão do cliente.

### Por que reimplementar `CpfValidator` na Lambda em vez de referenciar o assembly `Domain`

A Lambda vive em um repositório diferente (`tech-challenge-lambda`) do `Domain` do Atendimento (`Tech-challenge`). Referenciar o assembly inteiro do outro repositório criaria acoplamento entre repositórios que o desafio pede para serem independentes (cada um com seu próprio CI/CD). `CpfValidator` é uma função pura de poucas linhas — reimplementá-la é mais barato que o acoplamento cross-repo.

---

## Alternativas consideradas

### Alternativa 1: Autenticação via CPF dentro do próprio Atendimento (endpoint HTTP normal, sem Lambda)

**Prós:** sem reimplementação de `CpfValidator`, sem gerenciar um repositório/deploy adicional.

**Contras:** não atende ao requisito obrigatório do desafio, que pede explicitamente uma Function Serverless para essa validação.

**Por que não:** viola requisito obrigatório — não é alternativa viável, apenas registrada para mostrar que foi considerada.

### Alternativa 2: Cognito ou serviço de identidade gerenciado da AWS

**Prós:** solução totalmente gerenciada, sem código de autenticação para manter.

**Contras:** Cognito é desenhado para gerenciar identidades com senha/MFA/federação — o fluxo do desafio é mais simples (CPF já cadastrado pelo atendente, sem senha do cliente) e não se encaixa no modelo de usuário do Cognito sem trabalho de adaptação desnecessário.

**Por que não:** complexidade de integração maior que o problema exige; o desafio pede explicitamente uma function serverless *própria*, não um serviço de identidade de terceiros.

### Alternativa 3: JWT assinado com chave diferente da usada pelo Atendimento

**Prós:** isolamento entre os dois fluxos de autenticação (interno vs. cliente externo).

**Contras:** exigiria que o Atendimento validasse dois emissores/chaves diferentes, com lógica de autorização duplicada; complexidade sem benefício de segurança adicional real, já que ambos os fluxos emitem tokens para o mesmo sistema de rotas.

**Por que não:** simplicidade — uma única chave e um único middleware de validação no Atendimento, independente de qual fluxo emitiu o token.

---

## Decisão

Lambda .NET em `tech-challenge-lambda`, exposta via rota `POST /atendimento/auth/cpf` do API Gateway, reimplementando `CpfValidator`, consultando o RDS diretamente (Npgsql, sem EF Core completo) e assinando JWT com a mesma chave HMAC-SHA256 do Atendimento, injetada via GitHub Secrets em ambos os repositórios.

## Consequências

Ver critérios de aceite e passos detalhados em [CARD-29](../cards/05-fase3-aws/CARD-29-lambda-autenticacao.md) (implementação da Lambda) e [CARD-30](../cards/05-fase3-aws/CARD-30-api-gateway-integracao.md) (integração com o API Gateway e validação end-to-end).
