# CARD-43b — Roteiro do vídeo da Fase 4

**Tipo:** Documentação / Entrega
**Status:** Concluído — roteiro em `entrega/fase4/roteiro-video.md` (2026-10-13); gravação é do time
**Depende de:** CARD-40, CARD-41, CARD-42
**Bloqueia:** Submissão no portal
**Decisão arquitetural:** [ADR-021](../../adr/ADR-021-kubernetes-local-sem-aws-fase4.md) (Kind, sem Lambda), [ADR-022](../../adr/ADR-022-observabilidade-fase4-stack-otel-local.md) (observabilidade)
**Repositórios alvo:** `tech-challenge-docs`

---

## Contexto

O vídeo tem até 15 minutos e precisa mostrar, de forma visível (requisito E-07):

1. fluxo completo de uma OS passando pelos microsserviços;
2. execução do Saga Pattern e tratamento de falhas;
3. deploy automatizado de pelo menos um microsserviço com validação de testes;
4. monitoramento e rastreamento dos fluxos distribuídos.

## Decisões (2026-10-13)

- **Roteiro cronometrado** em `entrega/fase4/roteiro-video.md`: cada bloco com tempo, o que aparece na tela, o que dizer e o comando ou request exato. Meta de 13 minutos, com folga.
- **Ambiente:** o Kind local do `kind/subir.sh`, já no ar antes de gravar; a collection do CARD-43a dirige o fluxo.
- **Deploy:** mostrar um PR com o CI de um serviço (testes, gate de cobertura e `deploy-kind`), já executado, e o merge na `main` publicando a imagem `sha-<commit>`. Não esperar o CI ao vivo.
- **Falha e compensação com estado final verificável:** a compensação depois do pagamento (estorno e `Compensated`) e um alerta disparando com o runbook.
- **Sem Lambda e sem AWS:** explicar em uma frase, com o ADR-021.
- **Checklist antes de gravar:** cluster novo, collection verde, dashboards abertos, nada sensível na tela (senhas, tokens).

## Implementação (2026-10-13)

- [`entrega/fase4/roteiro-video.md`](../../entrega/fase4/roteiro-video.md): 7 blocos, 13 min 10 s, com tabela que liga cada pedido do enunciado ao bloco e à evidência na tela. Inclui o preparo do ambiente, o checklist antes de gravar e o que fazer se algo falhar.
- **Ensaiado contra o Kind local, pelas APIs:** as pastas 3, 4 e 5 da collection (Newman), a busca no Jaeger pela tag `correlation_id`, a consulta de logs do dashboard (logs dos três serviços, com `TraceId`), o alerta `DlqCrescendo` com a mensagem publicada na fila do OS e a proteção de branch (`enforce_admins`).
- **Não ensaiado na interface:** o clique no link TraceId do Grafana e o Publish pela tela do RabbitMQ; o equivalente pela API funcionou.

## Critérios de aceite

- [x] O roteiro cobre os quatro itens do enunciado, cada um com a evidência que aparece na tela.
- [x] A soma dos tempos fica em até 15 minutos.
- [x] O roteiro foi ensaiado contra o ambiente (comandos e requests funcionam como escritos). *(Pelas APIs; dois cliques de interface não ensaiados, ver acima.)*
