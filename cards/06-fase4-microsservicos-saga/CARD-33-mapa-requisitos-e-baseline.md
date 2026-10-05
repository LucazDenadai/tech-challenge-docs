# CARD-33 — Matriz do rubric e baseline da Fase 4

**Tipo:** Planejamento / Análise
**Status:** To Do
**Depende de:** —
**Bloqueia:** CARD-34, CARD-43
**Repositórios:** `Tech-challenge`, `tech-challenge-lambda`, `tech-challenge-infra-k8s`, `tech-challenge-infra-db`, `tech-challenge-docs`
**Decisão arquitetural:** [ADR-014](../../adr/ADR-014-limites-microsservicos-fase4.md)

---

## Contexto

As fases anteriores deixaram código, infraestrutura e documentação que podem ser reaproveitados, mas a Fase 4 altera limites de serviço, bancos e pipelines. Antes de migrar qualquer coisa, é necessário distinguir o que existe, o que está validado e o que apenas consta em README/card. Esta matriz será a fonte rastreável entre rubric, implementação, evidência e card responsável.

## Escopo

Inventariar requisitos obrigatórios e entregáveis do enunciado fornecido pelo time. Para cada requisito registrar: identificador, texto resumido, obrigatório/opcional, serviço responsável, repositório atual e alvo, situação (atendido/parcial/ausente/não verificado), evidência reproduzível, dependência e card de implementação.

Incluir explicitamente: três serviços independentes; propriedade de repositório/infra/banco; SQL e NoSQL; REST/mensageria; proibição de acesso cruzado ao banco; Saga e compensações; Mercado Pago; unit tests; BDD; 80% por serviço; análise de qualidade; pipelines independentes; Kubernetes; observabilidade; documentação/API; vídeo <=15 min; PDF final.

## Critérios de aceite

- [ ] Existe uma matriz única, versionada neste diretório, com todos os requisitos e entregáveis do enunciado.
- [ ] Cada requisito tem serviço dono, card, tipo de evidência e condição objetiva de conclusão.
- [ ] O estado atual está baseado em execução, configuração ou artefato verificável; texto de README sozinho é marcado como não verificado.
- [ ] Débitos da fase anterior que possam bloquear a Fase 4 (incluindo evidência pendente de observabilidade/entrega) estão identificados sem reabrir escopo não relacionado.
- [ ] Ambiguidades do enunciado são listadas separadamente para resolução em ADR/RFC, incluindo se SQL/NoSQL é requisito global ou por serviço.
- [ ] A sequência prioriza o fluxo mínimo obrigatório antes de otimizações ou refinamentos não exigidos.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Rastrear os requisitos da Fase 4

  Cenário: Requisito obrigatório possui evidência e responsável
    Dado que um requisito obrigatório foi extraído do enunciado
    Quando a matriz da Fase 4 é revisada
    Então o requisito deve ter um serviço responsável
    E um card de implementação ou validação
    E uma evidência reproduzível para demonstrar sua conclusão

  Cenário: Evidência ainda não verificada não é declarada como concluída
    Dado que um README afirma que uma capacidade está pronta
    Mas não há execução ou artefato que confirme o comportamento
    Quando o baseline é registrado
    Então o estado deve ser "não verificado"
    E a evidência pendente deve ser descrita
```

## Passos

1. Transcrever os requisitos do enunciado em linhas atômicas, preservando obrigatoriedade e limite temporal do vídeo.
2. Inspecionar cards existentes CARD-01–32, ADRs e repositórios de entrega; não assumir que status histórico prova o estado atual.
3. Classificar evidências e lacunas sem corrigir código neste card.
4. Publicar a matriz e apontar ambiguidades para CARD-34/35/36.

## Evidências

- `matriz-requisitos.md` com fonte, responsável, status, critério verificável e card associado.
- Links para workflows, testes, manifests, documentação e evidência de execução usados no baseline.