# CARD-11 — Docker: Dockerfile + docker-compose

**Tipo:** Infra  
**Status:** To Do  
**Depende de:** CARD-06, CARD-08  
**Bloqueia:** CARD-12, CARD-13

---

## Contexto

Atualizar o Dockerfile e o docker-compose para suportar os dois microserviços (Atendimento na porta 8080, Estoque na porta 8081) e o RabbitMQ. O ambiente local deve subir completo com um único `docker-compose up`.

---

## Critérios de aceite

- [ ] `docker-compose up` sobe todos os serviços sem erro
- [ ] Atendimento responde em `localhost:8080/swagger`
- [ ] Estoque responde em `localhost:8081/swagger`
- [ ] RabbitMQ Management acessível em `localhost:15672`
- [ ] PostgreSQL com dois schemas (`atendimento`, `estoque`) criados automaticamente
- [ ] Migrations aplicadas automaticamente na subida de cada serviço
- [ ] Healthchecks configurados para todos os serviços
- [ ] Nenhuma credencial hardcoded — tudo via `.env`

---

## Serviços no docker-compose

```yaml
services:
  postgres:         # porta 5432
  rabbitmq:         # portas 5672 e 15672
  atendimento:      # porta 8080, depende de postgres + rabbitmq
  estoque:          # porta 8081, depende de postgres + rabbitmq
```

---

## Dockerfile (padrão para ambos os serviços)

Multi-stage build:
- Stage `build`: `mcr.microsoft.com/dotnet/sdk:10.0`
- Stage `runtime`: `mcr.microsoft.com/dotnet/aspnet:10.0`
- Roda como usuário não-root `app`
- Expõe porta 8080

Cada serviço tem seu próprio Dockerfile em:
- `src/Atendimento/OficinaMecanica.Atendimento.API/Dockerfile`
- `src/Estoque/OficinaMecanica.Estoque.API/Dockerfile`

---

## Variáveis de ambiente (.env)

```
# PostgreSQL
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres123
POSTGRES_DB=oficinamecanica

# JWT
JWT_KEY=<chave-minimo-32-chars>
JWT_ISSUER=OficinaMecanica
JWT_AUDIENCE=OficinaMecanicaUsers

# RabbitMQ
RABBITMQ_HOST=rabbitmq
RABBITMQ_USER=guest
RABBITMQ_PASSWORD=guest

# Serviços
ESTOQUE_SERVICE_URL=http://estoque:8080

# SMTP (opcional localmente)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=
SMTP_PASSWORD=
```

---

## Passos

1. Criar/atualizar `Dockerfile` para Atendimento
2. Criar `Dockerfile` para Estoque
3. Atualizar `docker-compose.yml` com todos os serviços
4. Atualizar `.env.example` com todas as variáveis
5. Validar `docker-compose up --build` completo
6. Testar fluxo end-to-end: abrir OS → finalizar → checar baixa de estoque
