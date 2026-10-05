# ADR-015 — Ownership dos dados e infraestrutura da Fase 4

**Status:** Aceito — decisões de ownership e infraestrutura do CARD-34
**Data:** 2026-10-05
**Autores:** Time Tech Challenge — Fase 4
**Relaciona-se a:** [ADR-014](ADR-014-limites-microsservicos-fase4.md), [CARD-34](../cards/06-fase4-microsservicos-saga/CARD-34-limites-e-propriedade-dados.md)

---

## Contexto

O PDF da Fase 4 exige pelo menos três microsserviços independentes, cada um com repositório, infraestrutura e banco próprios, e proíbe o acesso direto de um serviço ao banco de outro. O enunciado não determina nomes de repositórios, proprietário do cadastro de clientes/filiais nem se cluster, rede e broker precisam ser exclusivos por serviço.

O baseline da Fase 3 tem Atendimento e Estoque no mesmo repositório de aplicação, schemas PostgreSQL separados na mesma instância RDS, um pipeline que publica os dois serviços juntos e uma Lambda que, segundo o README, consulta Clientes diretamente no RDS. Essas escolhas não podem ser carregadas para a arquitetura Fase 4 sem validar as novas regras de ownership.

Esta decisão fecha as opções necessárias para avançar ao CARD-35. Ela atende literalmente à separação de serviço, repositório, infraestrutura e banco, reutilizando a plataforma comum sem compartilhar bancos de dados. A alocação SQL/NoSQL permanece no CARD-35.

---

## Decisão

### Repositórios de serviço

Manter/criar um repositório de código por microsserviço de negócio, com estes nomes oficiais para o projeto:

| Microsserviço | Repositório definido | Unidade de código/deploy |
|---|---|---|
| OS | `tech-challenge-os` | API e lógica dona de clientes, veículos, filiais e ordens de serviço |
| Billing | `tech-challenge-billing` | API/consumidores e lógica de orçamento e pagamento |
| Operações | `tech-challenge-operacoes` | API/consumidores com módulos internos de Estoque e Execução |

Os repositórios existentes `tech-challenge-infra-k8s`, `tech-challenge-infra-db` e `tech-challenge-lambda` não contam como substitutos desses três serviços de domínio. Os repositórios de serviço ainda precisam ser criados; esse trabalho ocorre nos cards de implementação correspondentes. A Lambda permanece como adaptador técnico de autenticação, não é dona de dados de negócio e não conta para o mínimo de microsserviços.

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

`filialId` é uma dimensão de negócio, não apenas um rótulo para exibição:

1. **OS:** OS mantém o cadastro mestre de filiais e associa cada ordem a um `filialId` imutável durante o fluxo normal. Uma mudança de filial exigiria uma operação explícita, auditada e fora do fluxo inicial.
2. **Operações/Estoque:** confirmado pelo time, saldo, reserva e movimentação são identificados por `(filialId, pecaId)`. Disponibilidade e reserva de uma OS usam o estoque da filial associada à ordem; não há empréstimo implícito de estoque de outra filial.
3. **Billing:** orçamento, aprovação, pagamento e estorno mantêm a referência `filialId` da OS para auditoria, reconciliação e relatórios por filial; Billing não consulta o cadastro de filiais no banco OS.
4. **Contratos:** comandos e eventos do fluxo carregam `filialId`. OS valida a associação ao criar a ordem e os serviços consumidores validam a presença e persistem a referência local. Serviços não consultam diretamente o banco OS para resolver o ID.

Na migração da Fase 3, como os dados existentes não possuem filial, criar uma filial de migração única (`FILIAL-LEGADA`) no cadastro de OS e associar a ela todas as OS, peças/saldos e registros correlatos migrados. Filiais reais adicionais são cadastradas depois, sem reatribuir registros legados automaticamente.

O PDF não exige segregação de autorização entre filiais. `filialId` não será usado sozinho como controle de acesso: a autorização continua seguindo as roles existentes. Transferência de estoque entre filiais e mudança de uma OS de filial ficam fora do fluxo inicial; adicionar esses fluxos requer regra e compensação próprias.

### Infraestrutura por serviço e plataforma compartilhada

- Cada repositório de serviço mantém Dockerfile, manifests Kubernetes, configuração, pipeline e definição de provisionamento do seu banco e demais recursos sob seu ownership. Alterar, testar ou implantar um serviço não publica os outros.
- Cada serviço terá um recurso de banco gerenciado fisicamente dedicado, credenciais, migrations, backup e ciclo de vida exclusivos. Não se considera um schema isolado dentro da mesma instância suficiente para esta entrega.
- O tipo SQL/NoSQL e a alocação de stores por serviço ficam para o CARD-35; a decisão de isolamento físico não predetermina qual serviço usará cada tecnologia.
- Reaproveitar EKS, VPC, API Gateway, broker e backend de observabilidade como **plataforma compartilhada**. O PDF exige infraestrutura própria por serviço, mas não exige um cluster Kubernetes, VPC ou broker exclusivos por serviço. Cada serviço terá Deployment/Service/configuração, identidade/permissões, banco, manifests e deploy independentes.
- `tech-challenge-infra-k8s` permanece dono do provisionamento da plataforma compartilhada. Cada repositório de serviço é dono de seus recursos de aplicação e banco; interfaces/outputs da plataforma compartilhada são consumidos explicitamente e não transferem ownership dos dados.
- Terraform pode ser usado por continuidade com a Fase 3, mas não é requisito do PDF da Fase 4. A ferramenta para os novos bancos e recursos de serviço será registrada no CARD-41.

### Lambda de autenticação existente

Manter a Lambda e o fluxo de CPF existentes como adaptador técnico para não reescrever autenticação sem necessidade. Remover sua conexão direta ao banco OS: a Lambda consulta um endpoint interno autenticado do OS para validar/localizar o cliente e recebe somente os dados mínimos para emitir o JWT. OS continua dono dos dados; a Lambda tem credencial de API com escopo mínimo e não recebe credenciais SQL de OS. A Lambda não conta como um dos três microsserviços de negócio e sua integração é mantida no repositório próprio existente.

---

## Alternativas consideradas

### Compartilhar uma instância RDS entre serviços com schemas isolados

**Rejeitada:** o PDF exige banco próprio por serviço. Como o custo não é uma restrição para o time e a interpretação mais literal reduz risco de avaliação, cada serviço recebe um recurso de banco gerenciado fisicamente separado. A escolha do mecanismo relacional/não relacional permanece no CARD-35.

### Criar um cluster Kubernetes por serviço

**Rejeitada:** o PDF requer deploy em Kubernetes e infraestrutura própria, mas os entregáveis por serviço são Dockerfile, manifests e pipeline; não exige cluster físico exclusivo. Um EKS compartilhado com Deployment, permissões, banco e deploy isolados por serviço atende ao escopo sem fragmentar desnecessariamente a plataforma.

### Duplicar o cadastro de cliente/filial nos três serviços

**Rejeitada:** criaria múltiplas fontes de verdade e sincronização implícita. Serviços podem manter snapshots mínimos ligados a um ID, mas não cópias editáveis dos cadastros mestres.

### Criar um microsserviço/repositório adicional de catálogo/filiais

**Não selecionada:** não é necessária para o mínimo de três, aumenta o número de deployments e não há requisito explícito de domínio separado. Reavaliar apenas com necessidade funcional comprovada.

---

## Consequências

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
| Mais bancos e repositórios aumentam operação | Padronizar pipeline, observabilidade, backup e templates; custos foram aceitos pelo time. |
| Autorização por filial não é definida pelo rubric | Propagar e validar `filialId`; não adicionar segregação de acesso sem requisito de produto aprovado. |
| AWS/EKS pode estar desligado fora das janelas de demonstração | Validar o caminho local e agendar janela de provisionamento para smoke/deploy e evidências. |

---

## Estado de implementação

Esta ADR fecha as decisões arquiteturais do CARD-34. Repositórios, bancos, APIs e migração da Lambda ainda não foram implementados; seguem nos CARD-37 a CARD-41. O status aceito desta ADR significa que a arquitetura está definida, não que os componentes estejam entregues.

---

## Referências

- [ADR-014 — Limites dos microsserviços da Fase 4](ADR-014-limites-microsservicos-fase4.md)
- [CARD-34 — Limites e propriedade dos dados](../cards/06-fase4-microsservicos-saga/CARD-34-limites-e-propriedade-dados.md)
- [Matriz de requisitos da Fase 4](../cards/06-fase4-microsservicos-saga/matriz-requisitos.md)
- [Diagrama de componentes — Fase 4](../diagramas/diagrama-componentes-fase4.md)