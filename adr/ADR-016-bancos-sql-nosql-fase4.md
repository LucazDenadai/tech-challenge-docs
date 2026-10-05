# ADR-016 — Persistência SQL e NoSQL da Fase 4

**Status:** Aceito — arquitetura de persistência; implementação pendente nos cards de serviço
**Data:** 2026-10-05
**Autores:** Time Tech Challenge — Fase 4
**Relaciona-se a:** [ADR-014](ADR-014-limites-microsservicos-fase4.md), [ADR-015](ADR-015-ownership-e-infraestrutura-fase4.md), [CARD-35](../cards/06-fase4-microsservicos-saga/CARD-35-decisao-sql-nosql.md)

---

## Contexto

O PDF exige pelo menos um banco relacional (SQL) e um não relacional (NoSQL), e exige banco próprio por microsserviço. A redação não afirma que cada microsserviço deve usar as duas categorias. A interpretação adotada é que SQL e NoSQL devem existir na solução como um todo e que cada serviço deve ter seu próprio store exclusivo. Essa interpretação fica registrada como risco de leitura do enunciado; se a equipe docente confirmar que cada serviço precisa das duas categorias, esta ADR deve ser revista antes da implementação.

O time precisa conseguir subir e testar toda a solução localmente sem consumir recursos/custos de AWS. O host de desenvolvimento tem Docker Compose. A AWS oferece DynamoDB Local como imagem Docker oficial, sem acessar o serviço DynamoDB hospedado; suas diferenças de comportamento em relação ao serviço real precisam fazer parte da estratégia de testes.

---

## Decisão

Cada serviço de negócio terá um recurso PostgreSQL gerenciado fisicamente dedicado. Operações, que combina Estoque e Execução, terá adicionalmente uma tabela DynamoDB exclusiva para a capacidade de Execução:

| Serviço | Store relacional | Store não relacional | Fonte da verdade |
|---|---|---|---|
| OS | Instância PostgreSQL OS | — | PostgreSQL: clientes, veículos, filiais, OS e histórico |
| Billing | Instância PostgreSQL Billing | — | PostgreSQL: orçamentos, aprovações, pagamentos e reconciliação |
| Operações | Instância PostgreSQL Operações | Tabela DynamoDB de Execução | PostgreSQL: catálogo, saldo, reserva e movimentação de estoque. DynamoDB: fila e agregado de execução/diagnóstico/reparo. |

Instâncias, usuários/roles, credenciais, migrations, backups e pipelines de dados são exclusivos por serviço. Nenhum serviço lê ou grava store de outro serviço. AWS EKS, rede, API Gateway, broker e backend de observabilidade continuam compartilháveis conforme ADR-015.

### Responsabilidade do DynamoDB

A tabela DynamoDB persiste a visão operacional da execução, cujo formato e campos podem evoluir com diagnóstico e reparo: identificador da execução/OS, filial, estado da execução, datas, versão de concorrência, resumo de diagnóstico, etapas do reparo e referências a peças. Não armazena saldo, preço mestre de peças, dados de pagamento ou uma cópia editável da OS.

Os padrões de acesso que justificam o store são: recuperar a execução por ID; listar fila por filial e estado, ordenada por criação/prioridade; e atualizar a execução de forma condicional/versionada. O desenho final de chaves e índices é entregue com o modelo e os testes do CARD-38b, antes de criar a tabela definitiva.

O PostgreSQL de Operações continua sendo a única fonte da verdade para estoque. NoSQL não duplica quantidade disponível nem substitui transações de reserva/movimentação. Execução e estoque são módulos distintos do mesmo microsserviço e cada agregado tem um store dono.

### Consistência entre os dois stores de Operações

- Não existe transação distribuída entre PostgreSQL e DynamoDB.
- Cada mutação de estoque é transacionada somente no PostgreSQL; cada mutação do agregado de execução é transacionada somente no DynamoDB.
- A coordenação entre reserva e início/avanço de execução usa comandos e eventos idempotentes sob a Saga do CARD-36. Se uma etapa posterior falhar, executa-se a compensação definida pela Saga; nunca se simula rollback atômico entre os bancos.
- Eventos de cada store são publicados por uma outbox gravada atomicamente no mesmo store que a mutação. Para DynamoDB, o item da execução e o item de outbox usam `TransactWriteItems`; um dispatcher publica no broker e marca/retenta a outbox de forma idempotente. Para PostgreSQL, usa-se a outbox relacional.
- Consumidores deduplicam por event/command ID e propagam `osId`, `filialId` e correlation ID. Falha na publicação não perde a mutação e fica observável para retry/reconciliação.

### Desenvolvimento e testes locais — DynamoDB Local

DynamoDB Local será o modo local padrão. Será incluído como serviço de Docker Compose da solução, usando a imagem oficial `amazon/dynamodb-local`, `-sharedDb`, armazenamento em diretório/volume local e porta padrão 8000. A configuração do serviço recebe o endpoint pelo ambiente:

| Execução | Endpoint |
|---|---|
| Aplicação executada no host | `http://localhost:8000` |
| Aplicação dentro do Docker Compose | `http://dynamodb-local:8000` |

O SDK usa região e credenciais fictícias locais; tais valores não são credenciais AWS e não podem ser reutilizados no ambiente cloud. Testes de integração usam DynamoDB Local e fixtures de tabela próprias, permitindo rodar o fluxo sem criar recursos AWS. Dados locais podem usar volume persistente para desenvolvimento; testes automatizados devem iniciar com estado isolado/recriado.

**Custo local:** DynamoDB Local não chama DynamoDB AWS e não incorre em cobrança AWS por leitura, escrita, armazenamento ou transferência. Há apenas consumo local de CPU/disco e download inicial da imagem. **Cloud:** a tabela de produção será provisionada como recurso DynamoDB sob ownership de Operações, com capacidade sob demanda, criptografia, IAM restrito, backup/PITR e alarmes/custos monitorados. A cobrança AWS se aplica somente ao recurso hospedado.

**Limitações conhecidas do DynamoDB Local:** é destinado a desenvolvimento/teste e não substitui validação cloud; throughput/provisioned capacity e IAM gerenciado não são simulados, não há PITR, vários estados de criação de tabela são imediatos e conflitos transacionais reais podem não ser reproduzidos como no serviço hospedado. Assim, CI usa o emulador para comportamento funcional, e uma janela de homologação valida IAM, provisionamento, backup e integração real. Cenários de conflito não reproduzíveis no emulador são cobertos por teste controlado do handler e/ou validação cloud.

### Estudo de alternativas NoSQL

| Alternativa | Local sem conta AWS | Produção coerente com a plataforma existente | Avaliação |
|---|---|---|---|
| **DynamoDB + DynamoDB Local** | Imagem Docker oficial, Docker Compose, endpoint local; sem cobrança AWS; permite desenvolvimento offline | Serviço gerenciado AWS, provisionamento IAM/IaC e integração nativa com AWS | **Selecionada.** Atende à necessidade de execução local fácil e evita administrar Mongo em produção. Aceitamos validar diferenças do emulador em cloud. |
| **MongoDB Community + MongoDB em produção** | Imagem/container local simples, dados persistidos via volume | Exigiria escolher Atlas externo, operar MongoDB no Kubernetes ou avaliar DocumentDB; cada opção adiciona decisão operacional e de compatibilidade | **Não selecionada.** Continua alternativa válida se DynamoDB Local falhar no uso real ou se a equipe preferir manter o mesmo servidor Mongo em local e produção. |

A recomendação não é baseada em afirmar que Mongo é difícil localmente: ambos podem rodar localmente em Docker. DynamoDB foi escolhido porque o caminho local oficial funciona com Docker Compose e o destino gerenciado se alinha à AWS já existente. Se a imagem local ou o SDK .NET tornar o ciclo de desenvolvimento/CI instável, CARD-38b reabre a comparação antes de consolidar dependência de produção.

---

## Alternativas SQL consideradas

- **PostgreSQL gerenciado dedicado por serviço:** selecionado por compatibilidade com o modelo relacional existente, transações para OS/Billing/estoque e familiaridade operacional da equipe.
- **PostgreSQL compartilhado com schemas:** rejeitado para Fase 4, pois não demonstra banco próprio por serviço, conforme ADR-015.
- **Um motor SQL diferente por serviço:** não selecionado; não há requisito que justifique o custo de operação e a migração adicional.

---

## Consequências

### Positivas

- A solução contém SQL e NoSQL e cada microsserviço tem store exclusivo.
- O time pode desenvolver/testar o módulo NoSQL localmente sem credenciais nem cobrança AWS.
- PostgreSQL mantém as transações críticas de inventário; DynamoDB é aplicado a um agregado com consultas por filial/estado e formato de execução evolutivo.
- A topologia e as credenciais mantêm boundaries de dados explícitos.

### Negativas e mitigação

| Risco | Mitigação |
|---|---|
| DynamoDB Local não reproduz todos os comportamentos cloud | Cobrir lógica localmente e validar IAM, capacity, PITR e casos de concorrência em homologação AWS. |
| Operações mantém dois stores em bounded contexts diferentes | Não duplicar entidades/saldo; usar outbox, idempotência, Saga e traces correlacionados; incluir teste de recuperação e reconciliação. |
| Interpretação do requisito SQL/NoSQL pode ser mais rígida | Registrar a leitura global; confirmar com docente se possível e revisitar a ADR se for exigido ambos os tipos em cada microsserviço. |
| DynamoDB é pago na AWS | Usar ambiente cloud sob demanda, capacidade sob demanda, alarmes de billing e destruir/manter apenas no período necessário conforme CARD-41. |

---

## Referências

- [AWS — DynamoDB Local](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html)
- [AWS — Executar DynamoDB Local em Docker Compose](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.DownloadingAndRunning.html)
- [AWS — Limitações do DynamoDB Local](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.UsageNotes.html)
- [MongoDB — Instalar MongoDB Community em Docker](https://www.mongodb.com/docs/manual/administration/install-community-docker/)
- [CARD-35 — Decidir alocação dos bancos SQL e NoSQL](../cards/06-fase4-microsservicos-saga/CARD-35-decisao-sql-nosql.md)