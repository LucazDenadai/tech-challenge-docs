# CARD-08h — Estoque: docker-compose

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-08g  
**Bloqueia:** CARD-08i

---

## Contexto

Adicionar o serviço `estoque-api` ao `docker-compose.yml` existente, que já tem o serviço `atendimento-api`. O Estoque usa o mesmo PostgreSQL, mas schema separado (`estoque`). Porta exposta: 8081.

---

## Critérios de aceite

- [ ] Serviço `estoque-api` adicionado ao `docker-compose.yml`
- [ ] `docker-compose up estoque-api` sobe na porta 8081
- [ ] Variáveis de ambiente sem credenciais hardcoded (usar `.env`)
- [ ] Health check configurado

---

## Trecho a adicionar no docker-compose.yml

```yaml
estoque-api:
  build:
    context: .
    dockerfile: Dockerfile
    args:
      PROJECT: OficinaMecanica.Estoque.API
  ports:
    - "8081:8080"
  environment:
    - ConnectionStrings__DefaultConnection=${POSTGRES_CONNECTION_STRING}
    - ASPNETCORE_ENVIRONMENT=Development
  depends_on:
    postgres:
      condition: service_healthy
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
    interval: 30s
    timeout: 10s
    retries: 3
```

---

## Pontos de atenção

- O `Dockerfile` existente pode precisar aceitar um `ARG PROJECT` para construir projetos diferentes (Atendimento ou Estoque)
- Verificar se o `Dockerfile` atual é genérico ou específico para o Atendimento antes de reutilizar

---

## Passos

1. Verificar se o `Dockerfile` atual suporta build multi-projeto via ARG
2. Ajustar `Dockerfile` se necessário
3. Adicionar bloco `estoque-api` no `docker-compose.yml`
4. Adicionar variáveis no `.env.example`
5. `docker-compose up estoque-api` e validar resposta na porta 8081
