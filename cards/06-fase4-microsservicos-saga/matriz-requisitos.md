# Matriz de requisitos — Tech Challenge Fase 4

**Fonte normativa:** texto do PDF oficial da Fase 4, transcrito pelo time em 2026-10-05. Em caso de conflito, prevalece o PDF.
**Baseline local da aplicação:** branch `fix/atendimento-min-replicas-1`, commit `1667509` (`2026-08-30`). Arquivos locais `.claude/*` e o PDF não foram usados como evidência de implementação.
**Repositórios remotos consultados:** `tech-challenge-lambda`, `tech-challenge-infra-k8s` e `tech-challenge-infra-db`, branches `main`; metadados indicavam atualização em 2026-08-31.
**Limite da verificação:** inspeção estática de arquivos e workflows. Não foram executados testes, chamadas ao Mercado Pago, deploys, `terraform apply`, consultas ao estado da AWS ou verificações de branch protection. Portanto, documentação sozinha não comprova funcionamento ao vivo.

## Legenda de status

- **Parcial:** há implementação/configuração reutilizável, mas não cumpre todo o requisito da Fase 4.
- **Ausente no escopo inspecionado:** não foram encontrados artefatos de implementação nos arquivos/repositórios consultados.
- **Não verificado:** depende de execução, configuração externa ou estado remoto que não foi validado.
- **Planejado:** há card atribuído, mas ainda não há implementação Fase 4.

## Requisitos obrigatórios

| ID | Requisito do PDF | Status no baseline | Evidência atual | Destino / card |
|---|---|---|---|---|
| R-01 | Pelo menos 3 microsserviços independentes | **Parcial** | A solution local contém Atendimento e Estoque; `Tech-challenge` é um único checkout. Lambda de autenticação não substitui o domínio de execução. ADR-014 define OS, Billing e Operações (Estoque + Execução). | CARD-34, CARD-37, CARD-38, CARD-39 |
| R-02 | Repositório, infraestrutura e banco próprios por microsserviço | **Parcial** | Há repositórios separados de Lambda, infra EKS e infra RDS; não são três repositórios de microsserviços de negócio. A aplicação local usa dois `DbContext`/schemas, `atendimento` e `estoque`; o README do `infra-db` descreve um RDS compartilhado. | CARD-34, CARD-35, CARD-37–39, CARD-41 |
| R-03 | Ao menos um banco relacional e um não relacional | **Parcial no baseline; arquitetura decidida** | A Fase 3 usa PostgreSQL. ADR-016 adota três stores PostgreSQL dedicados e DynamoDB para o dossiê/fila de execução em Operações, com DynamoDB Local em Compose. Interpreta-se o requisito SQL+NoSQL como global à solução; o PDF não esclarece se ambos são exigidos por serviço. A implementação ainda está pendente. | CARD-35, CARD-37–39 |
| R-04 | REST síncrono quando necessário e mensageria assíncrona | **Parcial** | Código/configuração local inclui adapter HTTP em Atendimento e RabbitMQ/MassTransit para baixa de estoque. Não implementa ainda todos os contratos entre OS, Billing e Operações. | CARD-36–40 |
| R-05 | Nenhum serviço acessa diretamente o banco de outro | **Parcial / risco conhecido** | Atendimento consulta Estoque via HTTP e não por tabela; contudo, o README da Lambda descreve consulta direta de Clientes no RDS do Atendimento. Confirmar a implementação atual e remover ou justificar esse acesso no desenho da Fase 4. | CARD-34, CARD-37, CARD-39 |
| R-06 | Implementar Saga para fluxo distribuído de OS | **Ausente no baseline; estratégia decidida** | Não foram encontrados código/artefatos Saga nos fontes consultados. ADR-017 (aceito) define orquestração pelo OS. RabbitMQ, retry e DLQ isolados não comprovam Saga. | CARD-36, CARD-40 |
| R-07 | Compensação/rollback seguro em falhas | **Ausente no baseline; compensações decididas** | ADR-017 define um caminho de falha com compensação para cada etapa (cancelamento, liberação de reserva, estorno total) e o escopo do MVP. Compensações ainda não implementadas/validadas. | CARD-36, CARD-39b, CARD-40 |
| R-08 | Integração de pagamentos com Mercado Pago | **Ausente no escopo inspecionado** | Busca nos fontes locais não encontrou integração Mercado Pago/pagamentos. Nenhum dos READMEs dos três repositórios de fase 3 consultados descreve Billing/Mercado Pago. | CARD-39, CARD-39b |
| R-09 | Testes unitários em todos os microsserviços | **Parcial** | Há projetos xUnit unitários e de integração para Atendimento e Estoque. Não existem ainda os três serviços Fase 4 nem testes correspondentes para eles. | CARD-37–40 |
| R-10 | Pelo menos um fluxo completo testado com BDD | **Ausente no escopo inspecionado** | Não há arquivos `.feature` no diretório `tests` e os `.csproj` consultados não declaram SpecFlow/Reqnroll/Gherkin. Os exemplos Gherkin dos cards são especificação documental, não testes executáveis. | CARD-40 |
| R-11 | Cobertura mínima de 80% por serviço | **Não verificado / parcial** | `coverlet.runsettings` configura formato OpenCover e o workflow aponta o SonarCloud para `**/coverage.opencover.xml`, mas os comandos `dotnet test` inspecionados não passam `--collect:"XPlat Code Coverage"`; a geração/ingestão do relatório não está comprovada. Também não foi encontrado gate explícito >=80% separado por serviço. Percentuais do relatório de vulnerabilidades não provam cobertura por serviço nesta revisão. | CARD-41 |
| R-12 | Validação de qualidade via SonarQube ou similar no CI | **Parcial** | `.github/workflows/dotnet.yml` executa análise SonarCloud no repositório de aplicação. A análise estática está configurada, mas upload/resultado e eventual quality gate exigido pelo PR não foram validados nesta revisão. Lambda também tem workflow de build/teste; presença/resultado atual de gate de qualidade para ela não foram verificados. | CARD-41 |
| R-13 | Pipeline independente por microsserviço: build, teste, qualidade e deploy | **Parcial** | `Tech-challenge` tem um workflow que testa e publica/deploya Atendimento e Estoque juntos. Lambda e os repositórios de infraestrutura têm workflows próprios, mas Lambda não é um dos três serviços de domínio Fase 4. | CARD-34, CARD-41 |
| R-14 | Deploy automatizado de cada microsserviço em Kubernetes | **Parcial / não verificado ao vivo** | A aplicação tem manifests Kubernetes e job de deploy EKS para Atendimento e Estoque em conjunto. Não há três deployments Fase 4 independentes; nenhum rollout atual foi verificado. | CARD-37–39, CARD-41 |
| R-15 | Reusar ferramentas de observabilidade da Fase 3 | **Parcial / não verificado ao vivo** | Existem extensões OpenTelemetry e manifests locais Prometheus/Grafana/Loki/Promtail/Jaeger; infraestrutura remota da Fase 3 documenta Datadog. Traces/alertas do fluxo Fase 4 e a disponibilidade live não foram exercitados. | CARD-42 |

## Entregáveis

| ID | Entregável do PDF | Status no baseline | Evidência atual | Destino / card |
|---|---|---|---|---|
| E-01 | Um repositório Git para cada microsserviço | **Ausente para a arquitetura Fase 4** | Existem repositórios para aplicação, Lambda, infra-k8s, infra-db e docs, mas não os três repositórios de domínio OS/Billing/Operações. | CARD-34, CARD-37–39 |
| E-02 | Código-fonte, Dockerfile e manifests Kubernetes por microsserviço | **Parcial** | Há Dockerfiles/manifests para os dois serviços atuais no monorepo. Não há artefatos de OS/Billing/Operações independentes nos respectivos repositórios. | CARD-37–39, CARD-41 |
| E-03 | CI/CD por microsserviço | **Parcial** | Workflows remotos existem para Lambda e repositórios Terraform; workflow da aplicação empacota/deploya os dois domínios atuais juntos. | CARD-41 |
| E-04 | Evidência de cobertura (prints ou links no README) | **Não verificado / parcial** | Workflow coleta cobertura localmente e envia dados para SonarCloud; não se confirmou gate >=80% por serviço nem links/prints atuais nos READMEs. | CARD-41, CARD-43 |
| E-05 | Documentação arquitetural por serviço | **Parcial** | Há READMEs e documentação arquitetural central da Fase 3; não há documentação final dos três limites Fase 4. | CARD-34, CARD-43 |
| E-06 | Swagger ou Postman atualizado | **Parcial** | A aplicação tem Swagger e collection Postman dos serviços atuais. Não cobre Billing, Mercado Pago ou os novos contratos OS/Operações. | CARD-37b, CARD-39, CARD-43 |
| E-07 | Vídeo público/não listado de até 15 min com fluxo, Saga/falhas, deploy/testes e observabilidade | **Não verificado para a Fase 4** | CARD-32 contém requisitos/documentação de vídeo da fase anterior; nenhum vídeo foi validado contra o roteiro Fase 4 nesta revisão. | CARD-43 |
| E-08 | PDF final com participantes, links, vídeo, diagrama, Saga, divisão e tecnologias | **Não verificado para a Fase 4** | Há documentos de fases anteriores; não foi encontrada evidência de PDF Fase 4 submetido no portal. | CARD-43 |

## Decisões/ambiguidades para fechar

| Tema | O que o PDF deixa aberto | Encaminhamento |
|---|---|---|
| SQL + NoSQL | Não explicita se ambos são exigidos em cada serviço ou se basta haver pelo menos um de cada na solução | ADR-016 adota ao menos um de cada tipo na solução; confirmar com a equipe docente se possível e revisitar se houver orientação diferente. |
| Infraestrutura própria | Exige infraestrutura por serviço, mas não define se cluster/rede/broker devem ser físicos e exclusivos | CARD-34 define recursos isolados por serviço e justifica qualquer plataforma compartilhada; não assumir que Terraform ou AWS é exigência do PDF. |
| Saga | Permite orquestração ou coreografia | ADR-017 (aceito) adota orquestração pelo OS; contratos e compensações definidos no CARD-36. |
| Microsserviços sugeridos | OS, Billing e Execução são exemplos de responsabilidades; Estoque pode ser agrupado com Execução se o mínimo de três serviços independentes for atendido | ADR-014 registra a divisão aprovada: OS, Billing e Operações (Estoque + Execução). |
| Pagamento | Exige Mercado Pago, mas produto/API, fluxo, eventos, estorno e credenciais não estão especificados | Definir no CARD-39b após consultar documentação oficial e usar sandbox. |
| AWS/ Terraform | Não aparecem como requisito explícito da Fase 4 | São decisões de implementação/reuso da Fase 3, não critérios de aceite do rubric desta fase. |

## Próximas ações

1. CARD-34: confirmar ownership de dados/filial, nomear repositórios e delimitar recursos de infraestrutura próprios.
2. CARD-35: fechar a interpretação SQL/NoSQL e aprovar a topologia de persistência.
3. CARD-36: concluído — orquestração pelo OS confirmada; contratos e compensações definidos no ADR-017.
4. CARD-37–39: implementar os três serviços com testes e contratos independentes.
5. CARD-40–43: validar fluxo, gates, operação e evidências finais conforme os critérios acima.