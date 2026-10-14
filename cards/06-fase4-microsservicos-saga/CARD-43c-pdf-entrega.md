# CARD-43c — PDF de entrega da Fase 4

**Tipo:** Documentação / Entrega
**Status:** Em andamento — PDF gerado (2026-10-13); falta o link do vídeo
**Depende de:** CARD-40, CARD-41, CARD-42
**Bloqueia:** Submissão no portal
**Decisão arquitetural:** [ADR-021](../../adr/ADR-021-kubernetes-local-sem-aws-fase4.md) (Kind, sem Lambda), [ADR-022](../../adr/ADR-022-observabilidade-fase4-stack-otel-local.md) (observabilidade)
**Repositórios alvo:** `tech-challenge-docs`

---

## Contexto

O portal pede um PDF com participantes, links dos repositórios, link do vídeo, diagrama geral da arquitetura (serviços, bancos e comunicação), estratégia da Saga, justificativa da divisão dos serviços e tecnologias (E-08).

## Decisões (2026-10-13)

- **Padrão visual da Fase 2 (decidido pelo time):** capa escura "POSTECH / TECH CHALLENGE" com tabela de identificação, títulos em azul, diagramas com legenda, decisões principais em texto corrido e links no fim. Fonte: `Tech-challenge/docs/Tech_Challenge_Fase2_Entrega.pdf`.
- **Participante (confirmado pelo time):** Lucaz Gabriel Teixeira Denadai — RM37376.
- **Fonte em HTML** (`entrega/fase4/entrega-fase4.html`), convertido em PDF pelo Chromium headless. O HTML fica versionado; o PDF é gerado de novo quando o link do vídeo existir.
- **Conteúdo:** descrição e objetivos; diagrama geral; divisão dos serviços e bancos (SQL e NoSQL); Saga orquestrada e compensações; tecnologias; CI/CD e deploy no Kind; observabilidade; decisões principais (ADRs da Fase 4); execução; links (repositórios, docs, Postman, vídeo).
- **Link do vídeo:** campo marcado como pendente até a gravação.

## Implementação (2026-10-13)

- Fonte: [`entrega/fase4/entrega-fase4.html`](../../entrega/fase4/entrega-fase4.html). PDF: [`entrega/fase4/Tech_Challenge_Fase4_Entrega.pdf`](../../entrega/fase4/Tech_Challenge_Fase4_Entrega.pdf), 9 páginas A4, gerado pelo Edge headless.
- Seções: capa com identificação; descrição e objetivos; diagrama geral (serviços, bancos, RabbitMQ, Mercado Pago, observabilidade, Kind); divisão dos serviços e bancos com a justificativa de SQL e NoSQL; Saga orquestrada (por que orquestração, fluxo normal, falhas e compensações, garantias de mensageria); tecnologias; CI/CD; monitoramento; decisões principais; execução; links.
- Repositórios conferidos como públicos: `tech-challenge-os`, `-billing`, `-operacoes`, `-infra-k8s` e `-docs`.

### Para fechar

1. Gravar e publicar o vídeo (roteiro no CARD-43b).
2. Trocar `[LINK DO VÍDEO]` no HTML pelo link e, se for outro dia, a data da capa e do rodapé.
3. Gerar o PDF de novo:

   ```sh
   msedge --headless --no-pdf-header-footer --print-to-pdf=Tech_Challenge_Fase4_Entrega.pdf entrega-fase4.html
   ```

## Critérios de aceite

- [ ] O PDF tem todos os itens do enunciado; o link do vídeo é preenchido antes da submissão.
- [x] O diagrama mostra só o que existe nos manifests e contratos.
- [x] Os links abrem sem login (repositórios públicos).
- [x] Nenhum segredo ou dado pessoal de cliente aparece.
