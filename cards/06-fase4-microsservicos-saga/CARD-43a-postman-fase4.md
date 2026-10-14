# CARD-43a — Postman da Fase 4: fluxo da Saga nos três serviços

**Tipo:** Documentação / API
**Status:** To Do — escopo definido (2026-10-13)
**Depende de:** CARD-40, CARD-41, CARD-42
**Bloqueia:** Submissão no portal
**Decisão arquitetural:** [ADR-021](../../adr/ADR-021-kubernetes-local-sem-aws-fase4.md) (Kind, sem Lambda), [ADR-022](../../adr/ADR-022-observabilidade-fase4-stack-otel-local.md) (observabilidade)
**Repositórios alvo:** `tech-challenge-docs` (`postman/`)

---

## Contexto

A collection `tech-challenge-os` cobre só o OS e não passa pela Saga: os status mudam por mensagem, sem rota HTTP. O enunciado pede Swagger ou Postman atualizado (E-06), e o vídeo precisa de um jeito repetível de mostrar o fluxo pelas APIs.

## Decisões (2026-10-13)

- **Uma collection nova**, `tech-challenge-fase4.postman_collection.json`, com o mesmo roteiro do BDD (CARD-40c): rotas públicas dos três serviços, cada ação com o papel de quem a faria (atendente no OS, mecânico em Operações, cliente e Admin no Billing).
- **Pastas:** saúde; login; preparação (cliente, veículo, filial, peça, serviço e saldo); fluxo completo até `Completed`; compensação depois do pagamento (falha no reparo, estorno, `Compensated`); pagamento recusado (liberação da reserva); casos de erro (401, 403, 400, 422).
- **Espera pela Saga:** os efeitos chegam por mensagem. Os requests de consulta repetem até a condição (no máximo cerca de 30 s) pelo `setNextRequest` do runner; fora do runner, um envio manual mostra o estado do momento.
- **Token de cliente sem Lambda (ADR-021):** um pré-request assina o JWT no formato da Lambda (role `Cliente`, claim `cliente_id`) com `jwt_key`, como já faz a collection do OS e o BDD.
- **Environment do Kind:** `os_url`, `billing_url` e `operacoes_url` nas portas do `kind/subir.sh`; `senha_demo` e `jwt_key` vazios, com o comando `kubectl` que os lê dos Secrets.
- **Repetível no mesmo cluster:** cada execução abre OS novas; os saldos são comparados com o valor lido no início da execução.

## Critérios de aceite

- [ ] A collection roda inteira com o Newman contra o Kind local, sem falhas, e pode ser repetida no mesmo cluster.
- [ ] O fluxo completo termina em `Completed`, com OS `Finalizada`, pagamento `Aprovado`, execução `Concluida` e baixa de estoque.
- [ ] A compensação termina em `Compensated`, com pagamento `Estornado` e estoque igual ao inicial.
- [ ] O pagamento recusado termina em `Cancelled`, com a reserva liberada.
- [ ] Nenhum segredo fica versionado no environment.
- [ ] `postman/README.md` explica como preencher o environment e rodar.
