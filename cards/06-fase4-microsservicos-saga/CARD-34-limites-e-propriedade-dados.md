# CARD-34 — Fechar limites, repositórios e propriedade dos dados

**Tipo:** Arquitetura
**Status:** Concluído — decisões registradas em ADR-015; implementação fica nos cards de serviço
**Depende de:** CARD-33
**Bloqueia:** CARD-35, CARD-36, CARD-37, CARD-38, CARD-39, CARD-41
**Repositórios:** `tech-challenge-docs`, `tech-challenge-os`, `tech-challenge-billing`, `tech-challenge-operacoes`; plataforma compartilhada em `tech-challenge-infra-k8s`
**Decisão arquitetural:** [ADR-014](../../adr/ADR-014-limites-microsservicos-fase4.md), [ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md)

---

## Contexto

ADR-014 registra a decisão de ter três serviços: OS, Billing e Operações (Estoque + Execução). O requisito de independência inclui repositório, infraestrutura e banco próprios. ADR-015 agora fecha os nomes, ownership, bancos dedicados, plataforma compartilhada e integração da Lambda. O CARD-34 conclui essas decisões; os recursos e o código serão implementados nos cards 37 a 41.

Este card torna executável o ADR-014; não escolhe tecnologia de banco nem decide o estilo da Saga, que pertencem aos CARD-35 e CARD-36.

## Escopo

- Definir nome e responsabilidade dos repositórios de OS, Billing e Operações e política de branch/PR.
- Produzir mapa de bounded contexts, entidades/dados, APIs produtoras/consumidoras e eventos publicados/consumidos.
- Definir o significado de “infraestrutura própria” para o rubric: recursos, banco, manifests e deploy independentes por serviço, distinguindo-os da plataforma compartilhada.
- Registrar ownership de cliente/filial e resolver como a filial acompanha OS, estoque, cobrança e execução.
- Documentar migração dos limites atuais, inclusive Atendimento/Estoque no monorepo atual e consulta direta da Lambda ao banco de Clientes.

## Critérios de aceite

- [x] Nomes dos repositórios de OS, Billing e Operações estão definidos em ADR-015; criação dos repositórios está atribuída aos cards de serviço.
- [x] Ownership de cada dado de negócio tem um único serviço escritor e responsável.
- [x] Operações inclui Estoque e Execução em módulos distintos, com ownership de saldo e movimentações.
- [x] Contratos entre serviços não exigem credenciais nem conexão direta ao banco de outro serviço.
- [x] Cada serviço terá recurso de banco gerenciado fisicamente dedicado, credenciais e ciclo de migrations/backup próprios.
- [x] EKS, VPC, API Gateway, broker e backend de observabilidade são plataforma compartilhada; cada serviço terá recursos Kubernetes, permissões, pipeline e deploy independentes.
- [x] `filialId` tem uso definido como dimensão de negócio: associação imutável à OS, partição de estoque, referência financeira e correlação de mensagens; não é uma regra de autorização por si só.
- [x] Registros legados da Fase 3 são associados a uma filial única `FILIAL-LEGADA`; transferência de estoque entre filiais e mudança de filial da OS ficam fora do fluxo inicial. *Revisto em 2026-10-05: não há migração de dados da Fase 3 (emenda do ADR-015), então `FILIAL-LEGADA` não é criada.*
- [x] A Lambda permanece como adaptador e consultará OS por API interna autenticada, sem acesso direto ao banco OS.
- [x] A decisão está aceita e registrada em ADR-015 e no diagrama Fase 4; CARD-35 define as tecnologias/alocação SQL-NoSQL.

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

  Cenário: Autenticar cliente sem acesso cruzado ao banco
    Dado que a Lambda recebeu uma solicitação de autenticação por CPF
    Quando precisa verificar se o cliente existe e está ativo
    Então consulta o endpoint interno autenticado do serviço OS
    E não usa credenciais nem conexão SQL do banco OS

  Cenário: Propagar a filial nos serviços sem acessar o cadastro mestre
    Dado que OS criou uma ordem associada a um filialId
    Quando Billing e Operações recebem os comandos/eventos correspondentes
    Então ambos preservam o mesmo filialId em seus registros locais
    E nenhum deles consulta a tabela de filiais no banco OS

  Cenário: Não consumir estoque de outra filial implicitamente
    Dado que a OS pertence à filial A
    E a peça necessária só tem saldo na filial B
    Quando Operações valida disponibilidade e reserva a peça
    Então não usa o saldo da filial B para a OS da filial A
    E informa indisponibilidade conforme o contrato da Saga

  Cenário: Rejeitar comando sem filial
    Dado que um comando de criação de OS ou execução não contém filialId
    Quando o serviço valida o contrato
    Então rejeita o comando antes de alterar estado ou saldo
    E não infere uma filial a partir de outro banco de serviço
```

## Passos

1. Registrar repositórios, ownership, infraestrutura por serviço e plataforma compartilhada em ADR-015.
2. Mapear componentes/artefatos existentes para os três serviços e identificar a migração da Lambda.
3. Produzir diagrama de contexto e tabela de ownership.
4. Fixar a política de filial e impedir acesso SQL entre serviços.
5. Deixar tecnologias SQL/NoSQL para CARD-35 e Saga/contratos para CARD-36.

## Evidências

- [ADR-015](../../adr/ADR-015-ownership-e-infraestrutura-fase4.md) aceito com nomes, ownership e plataforma.
- [Diagrama de Componentes Fase 4](../../diagramas/diagrama-componentes-fase4.md) e tabela de ownership.
- Lista dos repositórios e recursos por serviço, incluindo a substituição da consulta direta da Lambda por API OS.