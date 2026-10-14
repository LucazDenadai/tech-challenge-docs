# CARD-43b — Roteiro do vídeo da Fase 4

**Tipo:** Documentação / Entrega
**Status:** To Do — escopo definido (2026-10-13)
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

## Critérios de aceite

- [ ] O roteiro cobre os quatro itens do enunciado, cada um com a evidência que aparece na tela.
- [ ] A soma dos tempos fica em até 15 minutos.
- [ ] O roteiro foi ensaiado contra o ambiente (comandos e requests funcionam como escritos).
