# CARD-36 — Contratos entre serviços e desenho da Saga

**Tipo:** Arquitetura / Integração
**Status:** To Do
**Depende de:** CARD-34, CARD-35
**Bloqueia:** CARD-37, CARD-38, CARD-39, CARD-40
**Repositórios:** `tech-challenge-docs` e repositórios dos três serviços
**Decisão arquitetural:** Criar ADR para a estratégia de Saga e contratos após aprovação

---

## Contexto

O fluxo obrigatório atravessa OS, Billing e Operações. Mensageria, retry e DLQ já existem no projeto, mas não constituem por si só uma Saga. É necessário definir quem coordena o processo, quais transições são autorizadas, como eventos são versionados e quais ações compensam cada etapa. A escolha entre orquestração e coreografia permanece aberta até a comparação deste card.

## Escopo

Definir contratos REST/eventos, correlation/causation IDs, versionamento, ordenação e compatibilidade; máquina de estados da Saga; persistência do estado; timeout/retry/idempotência; estratégia outbox/inbox ou equivalente; política de mensagens inválidas/DLQ; eventos de aprovação e notificação Mercado Pago; e compensações para falhas de cobrança, reserva/baixa de peça e início/cancelamento da execução.

Comparar orquestração e coreografia considerando legibilidade do fluxo, acoplamento, recuperação, evidência demonstrável, complexidade operacional e aderência ao time. Não assumir que pagamento aprovado pode ser desfeito sem integração de estorno no provedor.

## Critérios de aceite

- [ ] Diagrama de sequência mostra o fluxo normal da abertura à execução e conclusão.
- [ ] Diagrama de estados identifica responsável por cada transição e estado persistido da Saga.
- [ ] ADR compara orquestração e coreografia e justifica escolha com base no fluxo real e nos requisitos do time.
- [ ] Contratos identificam produtor, consumidores, payload versionado e política de compatibilidade.
- [ ] Cada comando/evento inclui identificador de correlação suficiente para rastrear o processo distribuído.
- [ ] Para cada etapa há timeout, política de retry, comportamento idempotente e destino para falha não recuperável.
- [ ] Cada efeito que possa ocorrer antes de falha tem compensação definida ou é explicitamente classificado como não compensável, com fluxo de recuperação manual.
- [ ] A aprovação de pagamento é distinguida de confirmação de webhook; callbacks duplicados ou fora de ordem não duplicam efeitos.
- [ ] A decisão é revisada por todos os responsáveis pelos repositórios antes do início da implementação.
- [ ] Escrever o fluxo usando o mermaid para facilitar a interpretacao.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Definir contratos resilientes para a Saga da OS

  Cenário: Correlacionar eventos de uma ordem
    Dado que uma OS inicia uma Saga
    Quando um serviço publica um evento de domínio
    Então o evento carrega o identificador de correlação da Saga
    E o serviço consumidor registra esse identificador junto ao resultado

  Cenário: Definir compensação para falha após aprovação de pagamento
    Dado que o pagamento foi confirmado pelo provedor
    E a etapa seguinte do fluxo falhou
    Quando o desenho de Saga é revisado
    Então existe uma ação explícita de compensação ou recuperação
    E a ação tem estado, política de repetição e evidência de resultado
```

## Passos

1. Escrever fluxo normal e matriz de transições, responsáveis e efeitos persistidos.
2. Comparar orquestração e coreografia em RFC; registrar opção selecionada em ADR após aprovação.
3. Definir schemas dos eventos, APIs e política de versionamento.
4. Mapear falhas antes/depois de cada efeito e associar compensação, timeout e recuperação.
5. Atualizar diagramas de sequência, componentes e fluxo de pagamento.
6. Gerar contratos iniciais (OpenAPI/AsyncAPI ou formato equivalente escolhido) para CARD-37 a CARD-39.

## Evidências

- ADR aceito para Saga, com alternativas e consequências.
- Diagramas de sequência/estado e catálogo de eventos.
- Contratos versionados e tabela etapa → falha → compensação → verificação.