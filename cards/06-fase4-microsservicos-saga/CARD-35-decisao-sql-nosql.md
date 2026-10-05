# CARD-35 — Decidir alocação dos bancos SQL e NoSQL

**Tipo:** Arquitetura / Dados
**Status:** Concluído — arquitetura de persistência registrada em ADR-016; criação/configuração dos stores fica nos cards de serviço
**Depende de:** CARD-33, CARD-34
**Bloqueia:** CARD-37, CARD-38, CARD-39, CARD-41
**Repositórios:** `tech-challenge-docs` e os três repositórios de serviço
**Decisão arquitetural:** [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md)

---

## Contexto

O enunciado exige banco próprio por serviço e pelo menos um banco relacional e um não relacional. Adotamos a leitura de que SQL e NoSQL são requisitos para a solução como um todo, não que cada serviço tenha ambos. A redação permite essa leitura, mas não esclarece explicitamente o alcance; a matriz registra o risco e a equipe pode confirmar com os docentes.

O uso de NoSQL precisa atender a um padrão real de consulta da fila e do dossiê de execução. O ambiente local deve subir via Docker Compose sem conta AWS nem custo AWS.

## Escopo

- Registrar a leitura do requisito SQL/NoSQL e o risco de interpretação.
- Comparar PostgreSQL, DynamoDB/DynamoDB Local e MongoDB Community/produção.
- Alocar persistência e fonte da verdade por serviço, incluindo a separação entre Estoque SQL e Execução NoSQL dentro de Operações.
- Definir outbox/idempotência, endpoint local, credenciais fictícias e diferenças de emulador/cloud.
- Definir backup, restauração e retenção para cada store no nível necessário ao MVP.

## Critérios de aceite

- [x] ADR-016 registra alternativas, critérios, decisão e a leitura global de SQL/NoSQL, deixando explícito que o PDF não especifica o alcance.
- [x] OS, Billing e Operações têm recurso relacional dedicado; Operações também tem tabela DynamoDB sob seu ownership.
- [x] A solução inclui PostgreSQL e DynamoDB conforme a leitura registrada.
- [x] DynamoDB está ligado à fila/dossiê de execução; não foi escolhido apenas para marcar requisito.
- [x] Estoque e Execução têm stores e agregados distintos; não há duplicação do saldo nem dual-write para a mesma fonte de verdade.
- [x] Eventos usam outbox no store de origem, idempotência e Saga para coordenar efeitos entre stores/serviços.
- [x] Desenvolvimento local do NoSQL usa DynamoDB Local em Docker Compose e não consome serviço AWS.
- [x] Diferenças do emulador e validações que exigem AWS real estão listadas.
- [x] Credenciais e backups são exclusivos por serviço/store; a implementação dos recursos fica nos cards de serviço.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Isolar persistência por serviço

  Cenário: Serviço conecta somente aos stores que possui
    Dado que o mapa de persistência foi aprovado
    Quando as configurações de cada serviço são revisadas
    Então cada serviço possui credenciais próprias
    E nenhuma credencial permite consultar ou gravar o banco de outro serviço

  Cenário: NoSQL tem uma responsabilidade verificável
    Dado que um caso de uso foi alocado a um store NoSQL
    Quando a decisão arquitetural é revisada
    Então o padrão de leitura ou escrita que justifica essa escolha está documentado
    E a estratégia de recuperação e consistência está descrita

  Cenário: Iniciar o NoSQL local sem chamar AWS
    Dado que Docker Compose está disponível
    Quando o perfil local da solução é iniciado
    Então DynamoDB Local responde no endpoint configurado
    E a aplicação usa somente credenciais fictícias locais
    E nenhuma chamada é enviada ao serviço DynamoDB da AWS

  Cenário: Não duplicar o saldo de estoque no documento de execução
    Dado que Operações mantém o saldo como dado relacional
    Quando uma execução é gravada no DynamoDB
    Então o documento contém apenas referências aos itens de peça
    E o saldo permanece consultado e atualizado somente no PostgreSQL de Operações
```

## Passos

1. Registrar decisão e comparação no ADR-016 e atualizar a matriz de requisitos.
2. Provisionar stores exclusivos nos repositórios OS, Billing e Operações seguindo CARD-37 a CARD-39.
3. Implementar DynamoDB Local em Compose no CARD-38b e o recurso AWS gerenciado pelo IaC de Operações.
4. Usar o outbox/idempotência definidos no CARD-36/40 para publicar eventos sem dual-write.
5. Validar backup, PITR e comportamentos específicos da AWS em homologação.

## Evidências

- [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md) aceito e matriz de comparação nele registrada.
- Diagrama de persistência por serviço e política de acesso.
- Plano de backup/restore e relatório de execução local/cloud, produzido nos cards de implementação.