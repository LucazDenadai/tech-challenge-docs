# CARD-07 — Testes de integração do Atendimento

**Tipo:** Test  
**Status:** To Do  
**Depende de:** CARD-06  
**Bloqueia:** nenhum neste épico

---

## Contexto

Os testes de integração sobem a API completa com um PostgreSQL real via Testcontainers. Cobrem os fluxos críticos end-to-end, validando que controllers, use cases, repositórios e banco funcionam juntos corretamente.

---

## Critérios de aceite

- [ ] `CustomWebApplicationFactory` usando Testcontainers PostgreSQL
- [ ] Testes cobrem os 5 fluxos críticos listados abaixo
- [ ] Cada teste limpa o banco antes de rodar (isolamento)
- [ ] `dotnet test` passa 100% verde
- [ ] Tempo total de execução abaixo de 2 minutos

---

## Fluxos críticos a cobrir

### 1. Ciclo completo de uma OS
```
POST /ordens-servico           → 201, retorna osId
GET  /ordens-servico/{id}/status → 200, status = Recebida
PUT  /ordens-servico/{id}/status → 200, status = EmDiagnostico
PUT  /ordens-servico/{id}/status → 200, status = AguardandoAprovacao
PUT  /ordens-servico/{id}/orcamento (aprovado: true) → 200, status = EmExecucao
PUT  /ordens-servico/{id}/status → 200, status = Finalizada
```

### 2. Listagem ordenada
```
Criar 4 OS em status diferentes
GET /ordens-servico → ordem: EmExecucao > AguardandoAprovacao > EmDiagnostico > Recebida
OS Finalizada e Entregue não aparecem na listagem
```

### 3. Aprovação de orçamento — caminho de erro
```
PUT /ordens-servico/{id}/orcamento quando status != AguardandoAprovacao → 422
```

### 4. Transição de status inválida
```
PUT /ordens-servico/{id}/status com transição ilegal → 422 com mensagem clara
```

### 5. CRUD de clientes com validação de documento
```
POST /clientes com CPF válido → 201
POST /clientes com mesmo CPF → 409 Conflict
POST /clientes com CPF inválido → 400
DELETE /clientes/{id} → soft delete, GET ainda retorna mas Ativo = false
```

---

## Estrutura

```
IntegrationTests/
├── Fixtures/
│   ├── CustomWebApplicationFactory.cs
│   ├── AuthHelper.cs
│   └── IntegrationTestCollection.cs
└── Controllers/
    ├── OrdensServicoControllerTests.cs
    └── ClientesControllerTests.cs
```

---

## Passos

1. Criar `CustomWebApplicationFactory` com Testcontainers
2. Criar `AuthHelper` para geração de tokens nos testes
3. Implementar os testes do ciclo completo da OS
4. Implementar os testes de listagem ordenada
5. Implementar os testes de validação e erros
6. Rodar `dotnet test` e garantir 100% verde
