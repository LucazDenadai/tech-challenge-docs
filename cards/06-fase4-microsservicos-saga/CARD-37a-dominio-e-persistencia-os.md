# CARD-37a — Extrair domínio e persistência de OS

**Tipo:** Implementação / Dados
**Status:** To Do
**Depende de:** CARD-34, CARD-35
**Bloqueia:** CARD-37b
**Repositório alvo:** Repositório exclusivo de OS (definido no CARD-34)
**Decisão arquitetural:** ADR-014 e decisão de bancos do CARD-35

---

## Contexto

O atual Atendimento precisa ser dividido sem manter dependência de implantação ou banco com Billing e Operações. Este subcard isola o agregado OS, suas regras, migrations e dados, preservando histórico e estados válidos. A estratégia de migração dos dados legados deve ser reversível até o corte definido.

## Critérios de aceite

- [ ] O projeto OS tem camadas e dependências alinhadas ao padrão existente, sem dependência de código em repositório remoto compilado como atalho.
- [ ] Agregado e regras de transição da OS residem em OS; dados de orçamento, pagamento, estoque ou execução não são modelados como dados gerenciáveis locais.
- [ ] Contexto e migrations apontam apenas para o banco OS e não criam schemas/tabelas de outros serviços.
- [ ] Migrations podem ser aplicadas a banco vazio e a uma cópia controlada dos dados legados, conforme plano aprovado.
- [ ] A estratégia de migração identifica fonte, transformação, validação, corte, rollback e proteção contra perda/duplicação.
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
3. Implementar migrations e testes; planejar cópia/corte somente após validar integridade.
4. Registrar totais e checksums/regras de reconciliação para confirmar consistência da migração.

## Evidências

- Testes de domínio e integração passando em CI.
- Plano de migração, scripts/versionamento e relatório de reconciliação sem dados pessoais reais.