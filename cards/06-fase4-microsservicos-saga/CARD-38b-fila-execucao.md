# CARD-38b — Fila e ciclo de execução da OS

**Tipo:** Implementação / Domínio
**Status:** To Do
**Depende de:** CARD-36, CARD-38a
**Bloqueia:** CARD-40
**Repositório alvo:** Repositório exclusivo de Operações
**Decisão arquitetural:** Contratos e máquina de estados aprovados no CARD-36

---

## Contexto

Operações precisa controlar trabalho físico sem transformar o estado de execução no estado da OS. Uma ordem pode estar em execução enquanto OS mantém seu próprio estado agregado. Progresso e conclusão são informados por eventos versionados, e a execução deve conservar filial, técnico/responsável e histórico necessário.

## Critérios de aceite

- [ ] Fila e estados de execução são definidos (por exemplo, aguardando, diagnóstico, aguardando peça, reparo, concluída, cancelada), com transições válidas registradas.
- [ ] Início depende de comando/evento aceito pelo contrato e não ocorre antes das condições de aprovação definidas.
- [ ] Atualizações incluem filial, timestamps e correlation ID conforme modelo aprovado.
- [ ] Evento de conclusão/falha só é publicado depois de persistência confirmada.
- [ ] Cancelamento/repetição não deixa execução órfã e segue compensações do CARD-36.
- [ ] API ou interface de operação oferece consulta de fila, detalhe e atualização autorizada.
- [ ] Testes cobrem transição inválida, evento repetido, timeout e conclusão.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Acompanhar execução da OS

  Cenário: Atualizar diagnóstico e progresso
    Dado que uma execução foi iniciada para uma OS aprovada
    Quando um técnico registra diagnóstico e avanço do reparo
    Então Operações registra as transições no seu banco
    E publica o progresso necessário ao serviço OS

  Cenário: Impedir início de execução sem autorização de fluxo
    Dado que a Saga ainda não liberou a ordem para execução
    Quando uma solicitação de início é recebida
    Então Operações não inicia o trabalho
    E devolve um resultado de rejeição correlacionado
```

## Passos

1. Definir estados, roles e regras junto ao contrato e desenho da Saga.
2. Implementar fila e casos de uso com testes antes da API.
3. Adicionar endpoints/eventos e trilha de auditoria.
4. Validar visualmente o fluxo pela API/Postman sem depender de query direta ao banco.

## Evidências

- Testes das transições e exemplos OpenAPI/Postman.
- Histórico de uma execução completa com correlation ID.