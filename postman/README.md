# Postman — Tech Challenge

Collection usada para gravar o vídeo de demonstração (CARD-32) e para validar o fluxo end-to-end sempre que a infraestrutura AWS é reativada (ver [ADR-009](../adr/ADR-009-migracao-aws-e-separacao-repositorios.md), seção de mitigação de custo).

## Arquivos

| Arquivo | Conteúdo |
|---|---|
| `tech-challenge.postman_collection.json` | 6 requests: healthcheck, login de atendente, criação de OS, login por CPF (Lambda), acompanhamento de OS sem/com token |
| `tech-challenge.postman_environment.json` | Variáveis com os valores usados na última janela de reativação — `base_url` do API Gateway, `cliente_id`/`veiculo_id` de teste |

## Como usar

1. Importar os dois arquivos no Postman (File → Import)
2. Selecionar o environment "Tech Challenge — AWS (janela de reativação)"
3. **Atualizar `base_url`** com o valor atual — o API Gateway gera uma URL nova a cada `terraform apply` do zero:
   ```bash
   cd tech-challenge-infra-k8s
   terraform output -raw api_gateway_url
   ```
4. Rodar as requests em ordem (1 → 2 → 3 → 4 → 5) — os scripts de teste preenchem `token_admin`, `token_cliente` e `numero_os` automaticamente a partir das respostas, sem copiar/colar manual

## Dados de teste usados

O cliente e veículo referenciados em `cliente_id`/`veiculo_id` foram criados manualmente na última janela de reativação:

| Campo | Valor |
|---|---|
| Cliente | "Cliente Demo Video", CPF `529.982.247-25` |
| Veículo | Fiat Uno, placa `DEM0123` |

Se a infra for destruída e reativada do zero (RDS recriado), esses registros deixam de existir — recrie um cliente/veículo via requests 1+2 do fluxo normal de atendente antes de rodar a collection, ou ajuste `cliente_id`/`veiculo_id` no environment.
