# CARD-34 — Fechar limites, repositórios e propriedade dos dados

**Tipo:** Arquitetura
**Status:** To Do
**Depende de:** CARD-33
**Bloqueia:** CARD-35, CARD-36, CARD-37, CARD-38, CARD-39, CARD-41
**Repositórios:** `tech-challenge-docs` e os três repositórios de serviço a identificar
**Decisão arquitetural:** [ADR-014](../../adr/ADR-014-limites-microsservicos-fase4.md)

---

## Contexto

ADR-014 registra a decisão já alinhada de ter três serviços: OS, Billing e Operações (Estoque + Execução). O requisito de independência inclui repositório, infraestrutura e banco próprios. Ainda é necessário mapear os repositórios existentes para esses limites, separar infraestrutura específica de serviço da plataforma compartilhada e especificar proprietário de cada dado, inclusive cliente, filial, peças, execução e pagamento.

Este card torna executável o ADR-014; não escolhe tecnologia de banco nem decide o estilo da Saga, que pertencem aos CARD-35 e CARD-36.

## Escopo

- Definir nome e responsabilidade dos repositórios de OS, Billing e Operações e política de branch/PR.
- Produzir mapa de bounded contexts, entidades/dados, APIs produtoras/consumidoras e eventos publicados/consumidos.
- Definir o significado de “infraestrutura própria” para o rubric: recursos e manifests implantáveis por serviço, sem duplicar cluster/plataforma compartilhada sem justificativa.
- Registrar ownership de cliente/filial e resolver como a filial acompanha OS, estoque, cobrança e execução.
- Documentar migração dos limites atuais, inclusive Atendimento/Estoque no monorepo atual e consulta direta da Lambda ao banco de Clientes.

## Critérios de aceite

- [ ] Cada serviço tem repositório de código, pipeline/deploy, definição de infraestrutura e banco designados.
- [ ] Existe tabela de ownership em que cada dado de negócio tem exatamente um serviço escritor e responsável.
- [ ] O modelo de Operações inclui Estoque e Execução, preservando módulos internos e ownership de saldo/movimentações.
- [ ] Nenhum fluxo proposto exige credenciais ou conexão a banco pertencente a outro serviço.
- [ ] Recursos compartilhados de plataforma (por exemplo, cluster, rede e broker) estão distinguidos dos recursos/manifestos próprios de cada serviço; a interpretação atende ao enunciado e é justificada.
- [ ] Estratégia de filial define ao menos como a filial é identificada, persistida e propagada em cada fluxo crítico, ou registra explicitamente uma pendência de negócio.
- [ ] ADR-014 e o diagrama de componentes são atualizados se o mapeamento aprovado alterar o limite já registrado.
- [ ] A consulta direta da Lambda ao banco de Clientes tem um plano de remoção ou uma exceção formal justificada, sem violar ownership.

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

1. Atualizar `matriz-requisitos.md` com os três serviços acordados.
2. Definir nomes dos repositórios e mapear componentes/artefatos existentes para origem e destino.
3. Produzir diagrama de contexto e tabela de ownership.
4. Escrever RFC/ADR para infraestrutura por serviço, filial e migração se restarem decisões transversais.
5. Revisar se ADR-014 precisa de atualização e registrar impactos em ADR-001, ADR-007 e ADR-009 sem editar o histórico.

## Evidências

- Diagrama C4 de contexto/componentes e tabela de ownership versionados em `diagramas/`.
- Lista de repositórios e recursos de infraestrutura por serviço, com consumidores dos outputs compartilhados.
- Revisão documentada da Lambda e do fluxo de filial.