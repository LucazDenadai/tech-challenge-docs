# ADR-010 — Sizing, custo e região da infraestrutura AWS

**Status:** Aceito
**Data:** 2026-07-18
**Autores:** Time Tech Challenge — Fase 3

---

## Contexto

O ADR-009 decidiu migrar para AWS (EKS, RDS, API Gateway, Lambda), mas não fixou região nem tamanho dos recursos. Diferente do Kind local (ADR-005, custo zero), a AWS cobra por hora ligada — e o EKS control plane não tem free tier, independente da idade da conta. É necessário decidir sizing e padrão de uso *antes* de escrever o Terraform que provisiona isso (CARD-27/28), para não haver risco de cobrança inesperada.

A conta AWS usada é antiga (sem free tier de 12 meses disponível) — todo o consumo é cobrado desde a primeira hora.

---

## Decisão

### Padrão de uso: sob demanda, não 24/7

A infraestrutura é provisionada (`terraform apply`) apenas quando necessário (desenvolvimento ativo, testes, gravação do vídeo de demonstração) e destruída (`terraform destroy`) ao final da sessão. Isso está alinhado ao comportamento já adotado na Fase 2 com Kind local, só que agora com custo real por hora ligada — deixar rodando 24/7 sem necessidade é desperdício direto.

### Região: us-east-1 (N. Virginia)

Escolhida em vez de `sa-east-1` (São Paulo) por: preço ~10-15% menor nos mesmos recursos, maior disponibilidade de tipos de instância, e maior volume de exemplos/documentação para os módulos `terraform-aws-modules/eks` e `/vpc` usados no CARD-28 — reduz risco de erro de configuração. A latência mais alta para acessos do Brasil é aceitável, já que o uso é para desenvolvimento/demonstração, não produção real com usuários finais.

### Sizing mínimo viável

| Recurso | Tamanho | Custo aproximado |
|---|---|---|
| EKS control plane | — (fixo) | ~US$ 0,10/hora |
| Nós do EKS | 2× `t3.small` | ~US$ 0,042/hora cada |
| RDS PostgreSQL | `db.t3.micro` | ~US$ 0,018/hora |
| API Gateway | — | cobrança por requisição, irrelevante no volume de testes |
| Lambda | — | cobrança por invocação, irrelevante no volume de testes |

**Total estimado: ~US$ 0,21/hora ligado** (~US$ 1,50 numa sessão de trabalho de 7 horas).

2 nós (não 1) foi escolhido como mínimo prático: um único nó arrisca falta de recursos para rodar os pods de sistema do EKS junto com Atendimento e Estoque simultaneamente, o que forçaria ajustes artificiais no HPA (ADR-003) só para caber em capacidade insuficiente.

### Terraform state remoto (S3 + DynamoDB)

Como o `terraform apply` passa a rodar também via pipeline de CI/CD (GitHub Actions, CARD-27), o state não pode ficar local — o runner `ubuntu-latest` é efêmero e não preserva arquivos entre execuções. Backend remoto S3 (armazenamento do `.tfstate`) + DynamoDB (lock, evita apply concorrente) é criado uma vez, manualmente, antes do primeiro `terraform apply` do pipeline.

---

## Alternativas consideradas

### Alternativa 1: Rodar 24/7 durante todo o desenvolvimento da Fase 3

**Prós:** mais conveniente, sem re-provisionar a cada sessão de trabalho.

**Por que não:** EKS control plane sozinho custa ~US$ 73/mês rodando o mês inteiro, fora nós e RDS — risco real de fatura alta por esquecimento, sem benefício proporcional para uma atividade acadêmica.

### Alternativa 2: 1 nó único (`t3.small` ou `t3.micro`)

**Prós:** reduz custo de nós pela metade.

**Por que não:** risco de faltar recursos para rodar Atendimento + Estoque + pods de sistema do EKS ao mesmo tempo, exigindo ajustes artificiais no HPA só para caber em capacidade insuficiente — o ganho de custo (~US$ 0,04/hora) não compensa o risco operacional.

### Alternativa 3: sa-east-1 (São Paulo)

**Prós:** menor latência para acessos feitos do Brasil.

**Por que não:** custo mais alto e menos exemplos específicos da região para os módulos Terraform usados — para desenvolvimento/demonstração, a latência mais alta de `us-east-1` é irrelevante.

### Alternativa 4: State local (sem backend remoto)

**Prós:** mais simples de configurar inicialmente.

**Por que não:** inviabiliza o `terraform apply` automático via pipeline (CARD-27) — o runner `ubuntu-latest` começa cada execução do zero, sem state anterior, o que quebraria o requisito de deploy automático via CI/CD.

---

## Consequências

### Positivas
- Custo previsível e baixo (~US$ 0,21/hora ligado), compatível com uso esporádico de desenvolvimento/demonstração
- Backend remoto permite `terraform apply` seguro via pipeline, com lock contra execuções concorrentes
- Sizing de 2 nós evita instabilidade por falta de recursos no cluster

### Negativas e mitigações

| Risco | Mitigação |
|---|---|
| Esquecer de rodar `terraform destroy` após uma sessão | Documentar o hábito no README de `tech-challenge-infra-k8s`/`infra-db`; considerar alerta de billing na AWS |
| Bucket S3/tabela DynamoDB do backend remoto precisam existir antes do primeiro apply | Criados manualmente uma única vez (fora do Terraform gerenciado, para evitar problema de "bootstrap" circular) |
| Sizing pode não suportar carga de demonstração com HPA escalando muito | Ajustar sizing pontualmente se necessário durante os testes do CARD-31/32, documentando a mudança |

---

## Referências

- [ADR-009 — Migração para AWS e Separação em Repositórios](ADR-009-migracao-aws-e-separacao-repositorios.md)
- [AWS EKS Pricing](https://aws.amazon.com/eks/pricing/)
- [AWS RDS Pricing](https://aws.amazon.com/rds/postgresql/pricing/)
- [terraform-aws-modules/eks](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)
- [terraform-aws-modules/vpc](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest)
