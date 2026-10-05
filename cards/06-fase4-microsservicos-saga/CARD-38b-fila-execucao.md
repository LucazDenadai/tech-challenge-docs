# CARD-38b — Fila e ciclo de execução da OS

**Tipo:** Implementação / Domínio
**Status:** To Do
**Depende de:** CARD-36, CARD-38a
**Bloqueia:** CARD-40
**Repositório alvo:** Repositório exclusivo de Operações
**Decisão arquitetural:** [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md); contratos e máquina de estados no CARD-36

---

## Contexto

Operações precisa controlar trabalho físico sem transformar o estado de execução no estado da OS. Uma ordem pode estar em execução enquanto OS mantém seu próprio estado agregado. Progresso e conclusão são informados por eventos versionados, e a execução deve conservar filial, técnico/responsável e histórico necessário.

O agregado da execução é persistido em DynamoDB. Para desenvolvimento e testes locais, DynamoDB Local sobe via Docker Compose, com endpoint configurável (`http://localhost:8000` no host ou `http://dynamodb-local:8000` entre containers), credenciais fictícias e estado de teste isolado. O deploy cloud usa a tabela DynamoDB do ambiente AWS; testes locais não podem depender dela.

## Critérios de aceite

- [ ] Fila e estados de execução são definidos (por exemplo, em diagnóstico, diagnosticada, aguardando, aguardando peça, reparo, concluída, cancelada), com transições válidas registradas. Execução diagnosticada só entra na fila após `ExecutionStartRequested` (ADR-017).
- [ ] Início depende de comando/evento aceito pelo contrato e não ocorre antes das condições de aprovação definidas.
- [ ] Atualizações incluem filial, timestamps e correlation ID conforme modelo aprovado.
- [ ] Evento de conclusão/falha só é publicado depois de persistência confirmada.
- [ ] Cancelamento/repetição não deixa execução órfã e segue compensações do CARD-36.
- [ ] API ou interface de operação oferece consulta de fila, detalhe e atualização autorizada.
- [ ] DynamoDB Local pode ser iniciado pelo Compose sem configurar uma conta/credencial AWS.
- [ ] A fila pode ser consultada por filial e estado, e a execução por identificador, segundo as chaves/índices definidos e testados.
- [ ] Alteração do agregado e registro da outbox DynamoDB são atômicos conforme ADR-016; a publicação no broker é idempotente e recuperável.
- [ ] Testes cobrem transição inválida, evento repetido, timeout e conclusão.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Acompanhar execução da OS

  Cenário: Registrar diagnóstico antes do orçamento
    Dado que OS solicitou o diagnóstico de uma ordem recém-aberta
    Quando um técnico registra as peças e serviços necessários
    Então Operações persiste a execução como diagnosticada, fora da fila de execução
    E publica DiagnosisCompleted com os itens para o serviço OS

  Cenário: Atualizar progresso do reparo
    Dado que uma execução foi iniciada para uma OS aprovada
    Quando um técnico registra avanço do reparo
    Então Operações registra as transições no seu banco
    E publica o progresso necessário ao serviço OS

  Cenário: Impedir início de execução sem autorização de fluxo
    Dado que a Saga ainda não liberou a ordem para execução
    Quando uma solicitação de início é recebida
    Então Operações não inicia o trabalho
    E devolve um resultado de rejeição correlacionado

  Cenário: Executar teste local sem chamar AWS
    Dado que DynamoDB Local está iniciado pelo Docker Compose
    E o endpoint local foi selecionado por configuração
    Quando os testes de persistência de execução rodam
    Então usam apenas DynamoDB Local com credenciais fictícias
    E não acessam o endpoint hospedado da AWS
```

## Passos

1. Definir estados, roles e regras junto ao contrato e desenho da Saga.
2. Implementar fila e casos de uso com testes antes da API.
3. Adicionar endpoints/eventos e trilha de auditoria.
4. Validar visualmente o fluxo pela API/Postman sem depender de query direta ao banco.

## Evidências

- Testes das transições e exemplos OpenAPI/Postman.
- Histórico de uma execução completa com correlation ID.