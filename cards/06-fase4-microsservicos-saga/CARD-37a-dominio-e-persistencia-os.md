# CARD-37a — Extrair domínio e persistência de OS

**Tipo:** Implementação / Dados
**Status:** To Do
**Depende de:** CARD-34, CARD-35
**Bloqueia:** CARD-37b
**Repositório alvo:** `tech-challenge-os`
**Decisão arquitetural:** [ADR-014](../../adr/ADR-014-limites-microsservicos-fase4.md), [ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md), [ADR-016](../../adr/ADR-016-bancos-sql-nosql-fase4.md) e [ADR-017](../../adr/ADR-017-saga-orquestrada-os-fase4.md) (status da OS)

---

## Contexto

O atual Atendimento precisa ser dividido sem manter dependência de implantação ou banco com Billing e Operações. Este subcard extrai o código do agregado OS e dos cadastros sob ownership do OS (clientes, veículos, filiais e usuários funcionários), com migrations próprias. Não há migração de dados da Fase 3: o banco começa vazio e recebe um seed de demonstração (emenda do ADR-015).

## Critérios de aceite

- [ ] O projeto OS tem camadas e dependências alinhadas ao padrão existente, sem dependência de código em repositório remoto compilado como atalho.
- [ ] Agregado e regras de transição da OS residem em OS; dados de orçamento, pagamento, estoque ou execução não são modelados como dados gerenciáveis locais.
- [ ] Status da OS segue o mapeamento do ADR-017 (`EmDiagnostico`, `AguardandoAprovacao`, `AguardandoPagamento`, `EmExecucao`, `Finalizada`, `Entregue`, `Cancelada`); `Recebida` sai e `AguardandoPagamento` entra.
- [ ] Contexto e migrations apontam apenas para a instância PostgreSQL dedicada de OS e não criam schemas/tabelas de outros serviços.
- [ ] Migrations podem ser aplicadas a banco vazio, e um seed idempotente cria filial, usuários (Admin, Atendente, Mecânico) e clientes/veículos de exemplo sem dados pessoais reais.
- [ ] Entidades da Fase 3 que pertencem a outros serviços (`Servico`, `ItemPeca`, `ItemServico`, orçamento, tempo de execução) não são extraídas para o OS.
- [ ] Testes cobrem regras de domínio, persistência, histórico e estados inválidos.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Persistência exclusiva do serviço OS

  Cenário: Aplicar migrations em banco vazio
    Dado um banco OS vazio provisionado para o serviço
    Quando as migrations do serviço são aplicadas
    Então apenas estruturas pertencentes a OS são criadas
    E o serviço consegue persistir e consultar uma OS

  Cenário: Rejeitar transição inválida
    Dado que uma OS está em um estado que não permite a transição solicitada
    Quando o caso de uso tenta aplicar a transição
    Então o estado e o histórico permanecem inalterados
    E o erro de domínio é observável
```

## Passos

1. Identificar entidades/casos de uso OS no repositório atual e listar dependências que pertencem a outros serviços.
2. Criar estrutura do repositório e modelo de domínio/persistência segundo CARD-35.
3. Implementar migrations, seed e testes.

## Evidências

- Testes de domínio e integração passando em CI.
- Seed versionado e migrations aplicadas em banco vazio, sem dados pessoais reais.
