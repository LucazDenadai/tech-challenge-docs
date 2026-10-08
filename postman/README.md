# Postman — Tech Challenge

| Collection | Fase | Alvo |
|---|---|---|
| `tech-challenge.postman_collection.json` | 3 | Atendimento + Estoque via API Gateway na AWS |
| `tech-challenge-os.postman_collection.json` | 4 | Serviço OS ([CARD-37b](../cards/06-fase4-microsservicos-saga/CARD-37b-api-e-contratos-os.md)), local ou em cluster |

## Fase 4 — Serviço OS

### Arquivos

| Arquivo | Conteúdo |
|---|---|
| `tech-challenge-os.postman_collection.json` | 29 requests em 7 pastas: saúde, login de funcionários, cadastros, abertura e consulta da OS, endpoint interno da Lambda, acompanhamento pelo cliente e transições manuais. Inclui os casos 400, 401, 403, 404 e 422. |
| `tech-challenge-os-local.postman_environment.json` | `base_url` local, issuer e audience do JWT. Os segredos (`senha_demo`, `api_key_interna`, `jwt_key`) vêm vazios. |

### Como usar

1. Subir o OS com o seed de demonstração (ver README de `tech-challenge-os`).
2. Importar os dois arquivos e preencher no environment os mesmos valores passados ao OS:

   | Variável | Valor do OS |
   |---|---|
   | `senha_demo` | `Seed__SenhaUsuarios` |
   | `api_key_interna` | `Interno__ApiKey` |
   | `jwt_key` | `Jwt__Key` |

3. Rodar a collection inteira, em ordem. Os scripts guardam tokens, `filial_id`, `cliente_id`, `veiculo_id`, `os_id` e `numero_os`. Cada execução abre uma OS nova e a cancela no fim, então pode ser repetida no mesmo banco.

Pela linha de comando, com o Newman:

```sh
newman run tech-challenge-os.postman_collection.json -e tech-challenge-os-local.postman_environment.json \
  --env-var senha_demo=... --env-var api_key_interna=... --env-var jwt_key=...
```

Execução de 2026-10-08 contra o `main` do OS: 29 requests, 67 asserções, nenhuma falha.

### Observações

- **Correlation ID:** cada request envia um `X-Correlation-Id` novo e confere que a resposta devolve o mesmo valor.
- **Token de cliente:** localmente não há Lambda de CPF. O request de acompanhamento assina no pré-request um JWT no formato dela (role `Cliente`, claim `cliente_id`) com `jwt_key`. Em ambiente com a Lambda, use o token que ela devolve.
- **Fora da collection:** os status entre `EmDiagnostico` e `Finalizada` mudam pela Saga ([ADR-017](../adr/ADR-017-saga-orquestrada-os-fase4.md)), sem rota HTTP. Por isso a entrega só é testada no caso de falha (`422`). O fluxo completo entra no [CARD-40](../cards/06-fase4-microsservicos-saga/CARD-40-saga-e-bdd-integrado.md).

## Fase 3 — Atendimento via API Gateway

Collection usada para gravar o vídeo de demonstração (CARD-32) e para validar o fluxo end-to-end sempre que a infraestrutura AWS é reativada (ver [ADR-009](../adr/ADR-009-migracao-aws-e-separacao-repositorios.md), seção de mitigação de custo).

### Arquivos

| Arquivo | Conteúdo |
|---|---|
| `tech-challenge.postman_collection.json` | 6 requests: healthcheck, login de atendente, criação de OS, login por CPF (Lambda), acompanhamento de OS sem/com token |
| `tech-challenge.postman_environment.json` | Variáveis com os valores usados na última janela de reativação — `base_url` do API Gateway, `cliente_id`/`veiculo_id` de teste |

### Como usar

1. Importar os dois arquivos no Postman (File → Import)
2. Selecionar o environment "Tech Challenge — AWS (janela de reativação)"
3. **Atualizar `base_url`** com o valor atual — o API Gateway gera uma URL nova a cada `terraform apply` do zero:
   ```bash
   cd tech-challenge-infra-k8s
   terraform output -raw api_gateway_url
   ```
4. Rodar as requests em ordem (1 → 2 → 3 → 4 → 5) — os scripts de teste preenchem `token_admin`, `token_cliente` e `numero_os` automaticamente a partir das respostas, sem copiar/colar manual

### Dados de teste usados

O cliente e veículo referenciados em `cliente_id`/`veiculo_id` foram criados manualmente na última janela de reativação:

| Campo | Valor |
|---|---|
| Cliente | "Cliente Demo Video", CPF `529.982.247-25` |
| Veículo | Fiat Uno, placa `DEM0123` |

Se a infra for destruída e reativada do zero (RDS recriado), esses registros deixam de existir — recrie um cliente/veículo via requests 1+2 do fluxo normal de atendente antes de rodar a collection, ou ajuste `cliente_id`/`veiculo_id` no environment.
