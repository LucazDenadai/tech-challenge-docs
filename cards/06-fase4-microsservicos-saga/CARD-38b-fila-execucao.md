# CARD-38b — Fila e ciclo de execução da OS

**Tipo:** Implementação / Domínio
**Status:** Implementado — branch `feat/card-38b-execucao-operacoes` em `tech-challenge-operacoes`, testes verificados localmente em 2026-10-08; aguardando PR e CI (CARD-41)
**Depende de:** CARD-36, CARD-38a
**Bloqueia:** CARD-40
**Repositório alvo:** `tech-challenge-operacoes`
**Decisão arquitetural:** [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md); contratos e máquina de estados no CARD-36

---

## Contexto

Operações precisa controlar trabalho físico sem transformar o estado de execução no estado da OS. Uma ordem pode estar em execução enquanto OS mantém seu próprio estado agregado. Início, conclusão e falha são informados ao OS por eventos versionados (o progresso intermediário fica consultável na API de Operações), e a execução deve conservar filial, técnico/responsável e histórico necessário.

O agregado da execução é persistido em DynamoDB. Para desenvolvimento e testes locais, DynamoDB Local sobe via Docker Compose, com endpoint configurável (`http://localhost:8000` no host ou `http://dynamodb-local:8000` entre containers), credenciais fictícias e estado de teste isolado. O deploy cloud usa a tabela DynamoDB do ambiente AWS; testes locais não podem depender dela.

## Ponto de partida (2026-10-08)

O CARD-38a deixou pronto em `tech-challenge-operacoes`:

- Consumidor, inbox, outbox e despachante. Para os comandos deste card, basta incluir `DiagnosisRequested` e `ExecutionStartRequested` em `CatalogoCanaisOperacoes` e tratar cada um em `ProcessarMensagemSagaUseCase`. O teste de contrato lista esses canais e os eventos de diagnóstico e execução como pendentes do 38b.
- A porta `IEstoqueParaExecucao` (`ConcluirConsumoAsync`, `RegistrarConsumoComFalhaAsync`), idempotente, para consumir a reserva na conclusão ou na falha.
- O catálogo de peças e serviços com preço de tabela, fonte do snapshot do `DiagnosisCompleted`.

A outbox do PostgreSQL serve ao Estoque. Os eventos da execução saem da outbox do DynamoDB (`TransactWriteItems`, ADR-016).

## Decisões de implementação (2026-10-08)

- **Estados da execução:** `EmDiagnostico` → `Diagnosticada` (fora da fila) → `NaFila` → `EmReparo` → `Concluida` ou `Falhou`. `DiagnosticoRejeitado` encerra no início. Os nomes seguem o diagrama de sequência (Diagnosing, Diagnosed, Queued/Started). Transição fora dessa ordem é recusada pelo domínio.
- **Uma execução por OS:** `DiagnosisRequested` cria a execução e gera o `executionId` usado nos contratos. Um segundo pedido para a mesma OS devolve a execução existente.
- **Diagnóstico:** o técnico (`Mecanico` ou `Admin`) registra peças e serviços pela API. Operações busca o preço vigente no catálogo e publica `DiagnosisCompleted` com o snapshot (emenda "Catálogo de serviços" do ADR-015). Item inexistente ou inativo devolve `422`. O técnico também pode rejeitar o diagnóstico com motivo (`DiagnosisRejected`). Operações rejeita sozinho quando a filial não opera estoque.
- **Início:** `ExecutionStartRequested` só move para `NaFila` uma execução `Diagnosticada` cuja reserva está ativa e é da mesma OS. Nesse caso publica `ExecutionStarted`. Em qualquer outro caso publica `ExecutionStartRejected` sem consumir peça. A porta `IEstoqueParaExecucao` ganha a consulta da reserva.
- **Progresso:** início do reparo e etapas ficam no agregado e na API, sem evento para o OS (emenda "Progresso da execução" do ADR-015).
- **Conclusão e falha:** primeiro consome a reserva pelo `IEstoqueParaExecucao` (PostgreSQL, idempotente). Depois grava o agregado e a outbox no DynamoDB. Se a segunda gravação falhar, a chamada pode ser repetida sem consumir de novo. Os eventos são `ExecutionCompleted` e `ExecutionFailed`, com as quantidades consumidas.
- **Tabela DynamoDB:** uma tabela (`DynamoDb:Tabela`, padrão `operacoes-execucoes`) com chave `PK`/`SK`:

  | Item | PK | SK | Para quê |
  |---|---|---|---|
  | Execução | `EXEC#<executionId>` | `EXEC` | Agregado, com `Versao` para concorrência otimista |
  | Execução da OS | `OS#<osId>` | `EXEC` | Garante uma execução por OS e localiza pelo `osId` |
  | Inbox | `INBOX#<messageId>` | `INBOX` | Deduplicação dos comandos de execução |
  | Outbox | `OUTBOX#<messageId>` | `OUTBOX` | Evento a publicar |

  O índice `GSI1` serve a fila por filial e estado (`FILIAL#<filialId>#STATUS#<status>`) e as mensagens pendentes da outbox (`OUTBOX#PENDENTE`). O atributo é removido quando a mensagem é publicada, e ela sai do índice.
- **Atomicidade:** cada comando ou ação grava a inbox (quando há), o agregado e a outbox em um `TransactWriteItems`, com condições de existência e versão (ADR-016). Um segundo despachante publica a outbox do DynamoDB, com o mesmo publicador da outbox do PostgreSQL.
- **Mensageria:** `DiagnosisRequested` e `ExecutionStartRequested` entram no catálogo de canais. O `ProcessarMensagemSagaUseCase` passa a escolher o store pelo canal: Estoque no PostgreSQL, Execução no DynamoDB.
- **Timeout:** timeout ou erro transitório do DynamoDB sobe para o consumidor refazer a mensagem. Se a primeira tentativa chegou a gravar, a condição da inbox faz a nova tentativa sair como duplicata, sem segundo efeito.
- **Local e testes:** `docker-compose.yml` no repositório com PostgreSQL, RabbitMQ e `amazon/dynamodb-local`. Endpoint, região e credenciais fictícias vêm de configuração. A tabela é criada no start só com `DynamoDb:CriarTabela=true` (desenvolvimento e testes). Na nuvem, ela vem da IaC do CARD-41. Os testes sobem DynamoDB Local pelo Testcontainers, com tabela própria por teste.
- **API:** `/operacoes/execucoes`. Consultas (fila por filial e estado, detalhe, por OS) para funcionários. Diagnóstico, início do reparo, etapas, conclusão e falha para `Mecanico` e `Admin`.
- **Cancelamento durante a execução:** o AsyncAPI não tem comando de cancelamento para Operações. Cancelar uma OS em `NaFila` ou `EmReparo` pela Saga fica com o CARD-40, que decide se cria um comando novo. Uma execução diagnosticada cujo orçamento não foi aprovado continua como histórico, sem entrar na fila (ADR-017).

## Implementação (2026-10-08)

- **Repositório:** `tech-challenge-operacoes`, em 6 commits, um por passo: domínio, contratos e casos de uso, DynamoDB, mensageria, API e README.
- **Roteamento por store:** `ProcessarMensagemSagaUseCase` manda os comandos de Estoque para a transação do PostgreSQL e os de Execução para o `TransactWriteItems` do DynamoDB. O consumidor, a DLQ e as retentativas são os mesmos.
- **Duas outboxes, um publicador:** o `PublicadorRabbitMq` saiu do despachante do 38a e serve aos dois despachantes. O da Execução lê os pendentes no `GSI1` e, ao publicar, remove os atributos do índice.
- **Execução nova por OS:** a unicidade vem do item `OS#<osId>` com condição de inexistência na mesma transação. Dois `DiagnosisRequested` simultâneos para a mesma OS: o segundo recebe conflito, o consumidor refaz e encontra a execução existente.
- **Início idempotente:** o agregado guarda a `idempotencyKey` do `ExecutionStartRequested`. Outra mensagem com a mesma chave é registrada na inbox sem novo evento.
- **Diagnóstico pela API:** a execução nasce do `DiagnosisRequested`, e o técnico registra os itens pela API. Até o OS existir (CARD-40), a demonstração e os testes publicam o comando direto no RabbitMQ ou chamam o caso de uso.
- **Compose:** verificado em 2026-10-08. O `docker compose up -d dynamodb-local` sobe o DynamoDB Local, e uma tabela é criada com credenciais fictícias, sem conta AWS.

### Pendências

- **Cancelamento durante a execução:** continua sem comando no AsyncAPI; está no escopo do [CARD-40](CARD-40-saga-e-bdd-integrado.md).
- **Réplicas do despachante do DynamoDB:** a mesma questão do 38a. Duas réplicas podem publicar o mesmo pendente, e o consumidor deduplica. Uma trava por item (atualização condicional antes de publicar) fica para o CARD-41.
- **Postman:** a collection de Operações, já com execução, fica para o CARD-43.

## Critérios de aceite

- [x] Fila e estados de execução são definidos (por exemplo, em diagnóstico, diagnosticada, aguardando, aguardando peça, reparo, concluída, cancelada), com transições válidas registradas. Execução diagnosticada só entra na fila após `ExecutionStartRequested` (ADR-017).
- [x] Início depende de comando/evento aceito pelo contrato e não ocorre antes das condições de aprovação definidas.
- [x] Atualizações incluem filial, timestamps e correlation ID conforme modelo aprovado.
- [x] Evento de conclusão/falha só é publicado depois de persistência confirmada.
- [ ] Cancelamento/repetição não deixa execução órfã e segue compensações do CARD-36. *(Repetição coberta; cancelamento de execução em andamento depende de comando novo, decidido no CARD-40.)*
- [x] API ou interface de operação oferece consulta de fila, detalhe e atualização autorizada.
- [x] DynamoDB Local pode ser iniciado pelo Compose sem configurar uma conta/credencial AWS.
- [x] A fila pode ser consultada por filial e estado, e a execução por identificador, segundo as chaves/índices definidos e testados.
- [x] Alteração do agregado e registro da outbox DynamoDB são atômicos conforme ADR-016; a publicação no broker é idempotente e recuperável.
- [x] Testes cobrem transição inválida, evento repetido, timeout e conclusão.

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
    E o progresso fica consultável na API de Operações
    E nenhum evento de progresso é publicado para o OS, que só recebe início, conclusão ou falha (ADR-018)

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

Registradas em 2026-10-08, branch `feat/card-38b-execucao-operacoes`:

| Evidência | Onde | Resultado |
|---|---|---|
| OpenAPI gerado | `docs/openapi/operacoes-v1.json` | 19 rotas e 25 operações; 9 de execução sob `/operacoes/execucoes` |
| Testes de domínio, casos de uso e contrato | `tests/OficinaMecanica.Operacoes.UnitTests` | 127 aprovados. Transições inválidas, comando repetido, timeout do DynamoDB após o consumo, conclusão e falha. O contrato cobre os 4 comandos e os 9 eventos de Operações sem pendentes |
| Testes de DynamoDB, mensageria e API (PostgreSQL, RabbitMQ e DynamoDB Local via Testcontainers) | `tests/OficinaMecanica.Operacoes.IntegrationTests` | 51 aprovados |
| Execução completa com correlation ID | `FluxoDaExecucao_DiagnosticoReservaInicioEConclusao` (mensageria) e `CicloPelaApi_DiagnosticoFilaReparoEConclusao` (API) | Diagnóstico, reserva, início no mesmo trace do comando, etapa e conclusão. Todos os eventos com o mesmo `correlationId`; histórico com as 5 transições e o responsável; sobra devolvida ao estoque |
| Compose com DynamoDB Local | `docker-compose.yml` | Tabela criada com credenciais fictícias, sem conta AWS |
| Postman | — | Pendente (CARD-43) |