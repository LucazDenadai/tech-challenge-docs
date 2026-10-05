# ADR-015 — Proposta de ownership dos dados e infraestrutura da Fase 4

**Status:** Proposto — aguardando aprovação do time
**Data:** 2026-10-05
**Autores:** Time Tech Challenge — Fase 4
**Relaciona-se a:** [ADR-014](ADR-014-limites-microsservicos-fase4.md), [CARD-34](../cards/06-fase4-microsservicos-saga/CARD-34-limites-e-propriedade-dados.md)

---

## Contexto

O PDF da Fase 4 exige pelo menos três microsserviços independentes, cada um com repositório, infraestrutura e banco próprios, e proíbe o acesso direto de um serviço ao banco de outro. O enunciado não determina nomes de repositórios, proprietário do cadastro de clientes/filiais nem se cluster, rede e broker precisam ser exclusivos por serviço.

O baseline da Fase 3 tem Atendimento e Estoque no mesmo repositório de aplicação, schemas PostgreSQL separados na mesma instância RDS, um pipeline que publica os dois serviços juntos e uma Lambda que, segundo o README, consulta Clientes diretamente no RDS. Essas escolhas não podem ser carregadas para a arquitetura Fase 4 sem validar as novas regras de ownership.

Esta decisão permanece **proposta** até revisão e aprovação do time. Ela fixa um caminho simples para executar o CARD-34, sem assumir que escolhas de repositório ou tecnologia são exigências literais do PDF.

---

## Proposta de decisão

### Repositórios de serviço

Criar um repositório de código por microsserviço de negócio, com estes nomes provisórios:

| Microsserviço | Repositório proposto | Unidade de código/deploy |
|---|---|---|
| OS | `tech-challenge-os` | API e lógica dona das ordens de serviço |
| Billing | `tech-challenge-billing` | API/consumidores e lógica de orçamento e pagamento |
| Operações | `tech-challenge-operacoes` | API/consumidores com módulos internos de Estoque e Execução |

Os repositórios existentes `tech-challenge-infra-k8s`, `tech-challenge-infra-db` e `tech-challenge-lambda` não contam como substitutos desses três serviços de domínio. A Lambda pode permanecer como componente técnico de autenticação, mas não é dona de dados de negócio nem é necessária para satisfazer a quantidade mínima de microsserviços.

### Ownership dos dados

| Dado/capacidade | Serviço escritor e fonte da verdade | Como os demais serviços referenciam |
|---|---|---|
| Cliente e veículo | OS | IDs estáveis e dados mínimos necessários enviados por contrato; sem consulta ao banco OS |
| Cadastro de filiais | OS | `filialId` propagado em comandos/eventos; demais serviços não mantêm cadastro mestre |
| Ordem, status geral e histórico da OS | OS | `osId`; mudanças feitas apenas por OS após receber resultado de contrato/evento |
| Orçamento, aprovação e estado financeiro | Billing | `osId`, `clienteId` e `filialId` como referências; Billing guarda apenas dados financeiros necessários e snapshots mínimos justificados |
| Pagamento, referência do provedor e estorno | Billing | Referência por `osId`/`pagamentoId`; nenhum outro serviço grava ou consulta as tabelas de Billing |
| Catálogo, saldo, reservas e movimentações de peças | Operações / módulo Estoque | Consultas síncronas ou eventos/IDs contratados; nenhum outro serviço replica saldo como dado gerenciável |
| Fila, diagnóstico, reparo e progresso da execução | Operações / módulo Execução | Referência por `osId`; OS recebe progresso/conclusão por contrato/evento |

Billing pode receber por contrato os dados mínimos de pagador necessários ao Mercado Pago, mas não se torna fonte da verdade do cadastro do cliente. Retenção, minimização e tratamento de PII devem ser definidos no desenho da integração do CARD-39.

### Filial

Proposta: OS mantém a identidade/cadastro mestre da filial e associa cada ordem a um `filialId` imutável. Operações mantém saldo e movimentações por `filialId`; Billing associa orçamento e pagamento à mesma referência. As referências são transportadas por contratos versionados, sem acesso às tabelas de OS. A aplicação de autorização por filial ainda precisa ser decidida conforme o requisito funcional do produto; não presumir isolamento por filial se ele não estiver no PDF ou for aprovado pelo time.

### Infraestrutura por serviço e plataforma compartilhada

- Cada repositório de serviço mantém Dockerfile, manifests Kubernetes, configuração, pipeline e definição/provisionamento dos recursos sob seu ownership, incluindo um banco isolado conforme a decisão de persistência do CARD-35.
- “Banco próprio” significa store exclusivo, credenciais exclusivas e ciclo de migração/backup sob o serviço. O tipo físico/lógico do store e a alocação SQL/NoSQL ficam para o CARD-35.
- A proposta é reaproveitar um cluster EKS, VPC, API Gateway e broker como **plataforma compartilhada**, pois o PDF não exige cluster físico por serviço. Cada serviço continua com Deployment/Service/configuração, política de acesso e deploy independentes.
- O repositório/plataforma de infraestrutura pode continuar provisionando recursos compartilhados. Recursos de dados específicos de cada serviço devem ser isolados e identificáveis; evitar RDS/schema/credenciais compartilhados entre os três serviços.
- Terraform pode ser usado por continuidade com a Fase 3, mas não é requisito do PDF da Fase 4. A ferramenta deve ser registrada no CARD-41 somente se escolhida.

### Lambda de autenticação existente

Proposta: manter a Lambda apenas se o fluxo de autenticação continuar necessário. Remover o acesso direto ao banco OS: obter os dados necessários por API/contrato publicado pelo serviço OS, ou substituir o fluxo por autenticação dentro do próprio OS se isso reduzir dependências sem alterar o rubric. A alternativa escolhida deve ser decidida antes do cutover e não pode introduzir leitura SQL cross-service.

---

## Alternativas consideradas

### Duplicar o cadastro de cliente/filial nos três serviços

**Não proposta:** criaria múltiplas fontes de verdade e sincronização implícita. Serviços podem manter snapshots mínimos ligados a um ID, mas não cópias editáveis dos cadastros mestres.

### Criar um microsserviço/repositório adicional de catálogo/filiais

**Não proposta neste momento:** não é necessário para o mínimo de três, aumenta o número de deployments e não há requisito explícito de domínio separado. Reavaliar apenas com necessidade funcional comprovada.

### Cluster Kubernetes exclusivo por serviço

**Não proposto:** o PDF exige infraestrutura por serviço, mas não afirma que a plataforma de cluster precisa ser exclusiva. Isolamento de banco, permissões, manifests e pipeline por serviço satisfazem a independência operacional com menos duplicação. Confirmar interpretação no CARD-34 caso a equipe docente espere isolamento físico.

---

## Consequências se aprovada

### Positivas

- Cada dado de negócio tem fonte da verdade identificável.
- Billing e Operações não precisam conectar aos bancos de outros serviços.
- O fluxo da filial é preservado sem criar um quarto microsserviço.
- Serviços podem ser publicados independentemente sobre a plataforma de Kubernetes existente.
- A consulta direta da Lambda ao RDS deixa de atravessar a fronteira de ownership.

### Riscos e trabalho decorrente

| Risco | Tratamento |
|---|---|
| Extração de Atendimento/Estoque exige migração e pode afetar consumidores existentes | Planejar cópia/cutover/reconciliação por card de serviço, mantendo contratos compatíveis e sem apagar dados antes da validação. |
| Mais bancos e repositórios aumentam operação | Padronizar pipeline, observabilidade, backup e templates; não compartilhar bancos para reduzir esforço. |
| Branch e autorização por filial ainda não têm regra detalhada no rubric | Definir a necessidade funcional separadamente; por ora documentar e propagar `filialId`, sem inventar regras de acesso. |
| AWS/EKS pode estar desligado fora das janelas de demonstração | Validar o caminho local e agendar janela de provisionamento para smoke/deploy e evidências. |

---

## Critérios para aceitar ou revisar a proposta

- O time aprova ou altera os nomes dos três repositórios.
- O mapa de ownership é validado pelo responsável por cada serviço.
- O time confirma a interpretação de infraestrutura por serviço versus plataforma compartilhada; se necessário, consulta a equipe docente.
- A Lambda não lê diretamente o banco OS depois do cutover.
- O desenho de filial está consistente nos contratos dos três serviços.
- O diagrama Fase 4 e o CARD-34 refletem a decisão aprovada.

---

## Referências

- [ADR-014 — Limites dos microsserviços da Fase 4](ADR-014-limites-microsservicos-fase4.md)
- [CARD-34 — Limites e propriedade dos dados](../cards/06-fase4-microsservicos-saga/CARD-34-limites-e-propriedade-dados.md)
- [Matriz de requisitos da Fase 4](../cards/06-fase4-microsservicos-saga/matriz-requisitos.md)
- [Diagrama de componentes — Fase 4](../diagramas/diagrama-componentes-fase4.md)