# CARD-35 — Decidir alocação dos bancos SQL e NoSQL

**Tipo:** Arquitetura / Dados
**Status:** To Do
**Depende de:** CARD-33, CARD-34
**Bloqueia:** CARD-37, CARD-38, CARD-39, CARD-41
**Repositórios:** `tech-challenge-docs` e os três repositórios de serviço
**Decisão arquitetural:** ADR/RFC a criar após a decisão do time

---

## Contexto

O enunciado exige que cada microsserviço tenha banco próprio e que a solução use pelo menos um banco relacional (SQL) e um não relacional (NoSQL). A redação não esclarece se cada serviço precisa ter ambos ou se basta que exista ao menos um de cada no conjunto. A interpretação proposta para avaliação é **um ou mais bancos de cada tipo na solução**, com pelo menos um banco por serviço; registrar essa leitura e qualquer confirmação oficial disponível antes da implementação.

Não escolher NoSQL só para “marcar a caixa”: a tecnologia deve servir a um dado/caso de uso real e não pode criar dependência de banco compartilhado.

## Escopo

- Registrar a interpretação normativa do requisito (global vs. por serviço) e fonte da interpretação.
- Comparar opções SQL/NoSQL por consistência, consulta, transações, operação, custo, backup/restore, observabilidade e compatibilidade com AWS/Kubernetes existente.
- Propor qual serviço usa qual persistência e por quê; se um serviço usar dois stores, definir qual é source of truth e como evitar dual-write inconsistente.
- Definir banco lógico e físico, credenciais, rede, backup, migrações e provisionamento por serviço.
- Produzir ADR aceito pelo time antes de provisionar recursos ou introduzir dependências.

## Critérios de aceite

- [ ] ADR/RFC registra alternativas, critérios, decisão, consequências e se a exigência NoSQL vale para solução ou para cada serviço.
- [ ] Cada microsserviço tem banco sob ownership exclusivo, com acesso concedido apenas ao serviço dono.
- [ ] O sistema inclui ao menos um store SQL e um NoSQL conforme a interpretação registrada.
- [ ] Cada store tem justificativa ligada ao padrão de acesso e aos requisitos funcionais, não apenas ao requisito de tecnologia.
- [ ] Se houver dois stores no mesmo serviço, o documento define fonte da verdade, sincronização, consistência eventual e recuperação de falha sem dual-write silencioso.
- [ ] Esquemas, migrações, backup/restore e retenção estão definidos no nível necessário para o MVP.
- [ ] Credenciais não são compartilhadas entre serviços nem versionadas.

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
```

## Passos

1. Confirmar interpretação da exigência NoSQL e registrá-la na matriz de requisitos.
2. Avaliar ao menos uma alternativa relacional já usada e duas opções NoSQL compatíveis com operação do time.
3. Selecionar a alocação mínima que satisfaz o rubric com custo/complexidade controlados.
4. Documentar conexão, ownership, migração, backup e recuperação.
5. Aprovar ADR antes de alterar Terraform, aplicações ou pipeline.

## Evidências

- ADR/RFC aceito e matriz de comparação.
- Diagrama de persistência por serviço e política de acesso.
- Plano verificável de backup/restore e migração.