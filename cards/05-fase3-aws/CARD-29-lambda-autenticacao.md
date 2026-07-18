# CARD-29 — Lambda de autenticação via CPF

**Tipo:** Serverless
**Status:** To Do
**Depende de:** CARD-28
**Bloqueia:** CARD-30
**Decisão arquitetural:** [ADR-009](../../adr/ADR-009-migracao-aws-e-separacao-repositorios.md)

---

## Contexto

Function Serverless (.NET) que recebe um CPF, valida o formato (reutilizar `CpfValidator` do Domain — mesma lógica já usada em `Tech-challenge`), consulta a existência e o status do cliente no RDS, e emite um JWT válido para consumo das APIs protegidas do Atendimento.

**Trust boundary:** a Lambda roda fora do domínio hexagonal da aplicação principal — não referencia o projeto `Domain` do outro repositório para não criar acoplamento entre repositórios via referência de projeto. A validação de CPF é reimplementada aqui (poucas linhas, evita dependência cross-repo de um assembly inteiro por uma função pura).

---

## Critérios de aceite

- [ ] Lambda recebe CPF via evento do API Gateway (`{ "cpf": "..." }`)
- [ ] CPF inválido (formato) retorna 400 sem consultar o banco
- [ ] CPF válido mas cliente inexistente retorna 404
- [ ] CPF válido, cliente existente mas inativo retorna 403
- [ ] CPF válido, cliente existente e ativo retorna 200 com JWT assinado (mesma chave/algoritmo HMAC-SHA256 usada em `Tech-challenge`, compartilhada via Secrets Manager)
- [ ] JWT emitido é aceito pelas rotas protegidas do Atendimento (validação end-to-end no CARD-30)
- [ ] Testes unitários cobrindo os 4 cenários acima (TDD, conforme regra global do projeto)
- [ ] Cold start medido e documentado no README (ordem de grandeza, não meta rígida)
- [ ] Conexão ao RDS via connection string em variável de ambiente (Secrets Manager), nunca hardcoded

---

## Estrutura de arquivos

```
tech-challenge-lambda/
├── src/
│   └── OficinaMecanica.Auth.Lambda/
│       ├── Function.cs           ← handler
│       ├── CpfValidator.cs       ← reimplementação mínima
│       ├── ClienteRepository.cs  ← consulta direta ao RDS
│       └── JwtGenerator.cs
├── tests/
│   └── OficinaMecanica.Auth.Lambda.Tests/
└── template.yaml (ou serverless.yml) ← definição da função + permissões IAM
```

---

## Passos

1. Criar projeto .NET Lambda (`dotnet new lambda.EmptyFunction` ou equivalente)
2. TDD: escrever testes para os 4 cenários (CPF inválido, não encontrado, inativo, sucesso)
3. Implementar `CpfValidator` (reimplementação mínima, sem dependência do assembly Domain de `Tech-challenge`)
4. Implementar consulta ao RDS (Npgsql direto ou Dapper — sem trazer EF Core inteiro para a Lambda por peso/cold start)
5. Implementar geração de JWT compatível com o middleware de auth do Atendimento
6. Configurar permissões IAM mínimas da Lambda (acesso ao RDS via VPC, acesso ao Secrets Manager)
7. Testar via `sam local invoke` ou similar antes do deploy
8. Documentar no README: payload de entrada/saída, variáveis de ambiente exigidas, cold start observado
