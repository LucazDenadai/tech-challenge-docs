# CARD-34 — Fechar limites, repositórios e propriedade dos dados

**Tipo:** Arquitetura
**Status:** Proposta registrada — aguardando aprovação do time
**Depende de:** CARD-33
**Bloqueia:** CARD-35, CARD-36, CARD-37, CARD-38, CARD-39, CARD-41
**Repositórios:** `tech-challenge-docs` e os três repositórios de serviço a identificar
**Decisão arquitetural:** [ADR-014](../../adr/ADR-014-limites-microsservicos-fase4.md); proposta detalhada em [ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md)

---

## Contexto

ADR-014 registra a decisão já alinhada de ter três serviços: OS, Billing e Operações (Estoque + Execução). O requisito de independência inclui repositório, infraestrutura e banco próprios. A proposta de nomes, ownership, recursos específicos e plataforma compartilhada está em ADR-015; o estado permanece aguardando aprovação do time, pois esses detalhes não são determinados pelo PDF.

Este card torna executável o ADR-014; não escolhe tecnologia de banco nem decide o estilo da Saga, que pertencem aos CARD-35 e CARD-36.

## Escopo

- Definir nome e responsabilidade dos repositórios de OS, Billing e Operações e política de branch/PR.
- Produzir mapa de bounded contexts, entidades/dados, APIs produtoras/consumidoras e eventos publicados/consumidos.
- Definir o significado de “infraestrutura própria” para o rubric: recursos, banco, manifests e deploy independentes por serviço, distinguindo-os da plataforma compartilhada.
- Registrar ownership de cliente/filial e resolver como a filial acompanha OS, estoque, cobrança e execução.
- Documentar migração dos limites atuais, inclusive Atendimento/Estoque no monorepo atual e consulta direta da Lambda ao banco de Clientes.

## Critérios de aceite

- [x] Há nomes candidatos distintos dos repositórios de OS, Billing e Operações documentados em ADR-015; criação dos repos depende da aprovação.
- [x] Existe tabela proposta de ownership em que cada dado de negócio tem exatamente um serviço escritor e responsável.
- [x] O modelo proposto de Operações inclui Estoque e Execução, preservando módulos internos e ownership de saldo/movimentações.
- [x] Nenhum fluxo proposto exige credenciais ou conexão direta ao banco de outro serviço.
- [x] Recursos de plataforma compartilhados estão distinguidos dos recursos/manifests/deploy próprios; a proposta e sua relação com o PDF estão justificadas.
- [x] A proposta de filial define OS como dono do cadastro mestre e `filialId` como referência propagada; regras de autorização por filial ficam identificadas como pendência funcional.
- [x] Plano para eliminar acesso direto da Lambda ao banco OS está documentado em ADR-015.
- [ ] O time aprovou nomes, ownership, filial e interpretação de infraestrutura compartilhada.
- [ ] ADR-014/diagrama final refletem as decisões aprovadas, após revisão do time.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Propriedade dos dados entre microsserviços

  Cenário: Atualizar OS sem acessar o banco de Billing
    Dado que Billing confirmou um pagamento por contrato publicado
    Quando OS processa essa confirmação
    Então OS atualiza apenas o estado persistido em seu próprio banco
    E não abre conexão com o banco de Billing

  Cenário: Executar uma ordem na filial responsável
    Dado que uma OS está associada a uma filial
    Quando o serviço Operações recebe a solicitação de execução
    Então a filial deve permanecer identificável na execução
    E o dado deve ser persistido somente no banco de Operações
```

## Passos

1. Registrar a proposta de repositórios, ownership e infraestrutura em ADR-015.
2. Mapear componentes/artefatos existentes para os três serviços e identificar a migração da Lambda.
3. Produzir diagrama de contexto e tabela de ownership.
4. Revisar a proposta com os responsáveis e aprovar/ajustar nomes, propriedade de filial e plataforma compartilhada.
5. Atualizar ADR-014 e diagrama final após aprovação, sem reescrever ADRs históricos.

## Evidências

- [ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md) com proposta e estado de aprovação.
- [Diagrama de Componentes Fase 4](../../diagramas/diagrama-componentes-fase4.md) e tabela de ownership.
- Lista candidata de repositórios/recursos e plano de remoção do acesso direto da Lambda ao banco OS.