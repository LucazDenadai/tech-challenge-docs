# CARD-43 — Documentação e entrega final da Fase 4

**Tipo:** Documentação / Entrega
**Status:** To Do — dividido em 43a, 43b, 43c e 43d (2026-10-13)
**Depende de:** CARD-33, CARD-37, CARD-38, CARD-39, CARD-40, CARD-41, CARD-42
**Bloqueia:** Submissão no portal
**Repositórios:** OS, Operações, Billing e `tech-challenge-docs`
**Decisão arquitetural:** ADR-014 e decisões aprovadas nos CARD-35/36

---

## Contexto

O entregável precisa permitir que avaliadores reconstruam a arquitetura, executem as APIs e confirmem o fluxo sem credenciais ou explicações privadas do time. A entrega exige um vídeo público/não listado de até 15 minutos e um PDF único no portal. CARD-32 é material da Fase 3 e não substitui a evidência específica da Fase 4.

## Escopo

- README autocontido em cada repositório de microsserviço com objetivo, ownership, arquitetura, tecnologias, pré-requisitos, configuração segura, teste, execução, deploy, Swagger e evidências.
- Diagramas finais de contexto/componentes, bancos, comunicação, Saga/compensações e observabilidade.
- Swagger/OpenAPI ou Postman atualizada cobrindo endpoints, autenticação, fluxo feliz e respostas de falha.
  - Já existem a collection do OS ([CARD-37b](CARD-37b-api-e-contratos-os.md), `postman/`) e o OpenAPI de Operações (`docs/openapi/operacoes-v1.json`, [CARD-38a](CARD-38a-estoque-em-operacoes.md)). Falta a collection de Operações e a do fluxo completo.
- Cobertura por serviço >=80%, workflow de qualidade e links/artefatos reproduzíveis.
- Vídeo de até 15 minutos mostrando fluxo completo, Saga e compensação, deploy automático de ao menos um serviço com validação de testes, monitoramento e trace distribuído.
- PDF único no portal com participantes, links dos três repositórios e docs, link do vídeo, diagrama geral, descrição/justificativa da Saga, divisão de serviços e tecnologias.

## Subcards

- [CARD-43a — Postman da Fase 4: fluxo da Saga nos três serviços](CARD-43a-postman-fase4.md)
- [CARD-43b — Roteiro do vídeo da Fase 4](CARD-43b-roteiro-video.md)
- [CARD-43c — PDF de entrega da Fase 4](CARD-43c-pdf-entrega.md) (padrão visual da Fase 2)
- [CARD-43d — READMEs dos serviços e revisão final da entrega](CARD-43d-readmes-revisao-final.md)

## Pendências vindas de outros cards

- **Sem AWS ([ADR-021](../../adr/ADR-021-kubernetes-local-sem-aws-fase4.md)):** a Lambda de CPF não roda na Fase 4. Definir como o vídeo obtém o token de cliente (mesmo formato da Lambda, como no BDD) e explicar a decisão. *(Decidido no CARD-43a: pré-request do Postman assina o JWT no formato da Lambda.)*

## Critérios de aceite

- [ ] Os READMEs dos três serviços explicam propriedade de dados, contrato e como executar testes sem acesso a produção.
- [ ] Diagramas concordam com os manifests, contratos e ADRs implementados; não exibem serviços/infra inexistentes.
- [ ] Collection/OpenAPI foi exercitada contra a versão de demonstração ou ambiente local equivalente.
- [ ] Links para cobertura e quality gates mostram resultado separado por serviço e limite >=80%.
- [ ] O roteiro de vídeo contém os quatro itens expressamente pedidos e termina dentro de 15 minutos.
- [ ] Vídeo demonstra ao menos um cenário de compensação com estado final verificável, não apenas a arquitetura em slides.
- [ ] PDF contém todos os itens do enunciado e links acessíveis; identificações/participantes são revisados pelo time.
- [ ] Revisão final verifica que nenhum segredo, CPF real, cartão ou dado pessoal de cliente foi publicado.
- [ ] Submissão é registrada somente após abrir/validar os links e confirmar upload no portal.

## Cenários de aceite (Gherkin)

```gherkin
Funcionalidade: Entregar evidências reproduzíveis da Fase 4

  Cenário: Avaliador encontra o fluxo documentado
    Dado que o avaliador acessa o README de cada microsserviço
    Quando segue os links de arquitetura e APIs
    Então identifica responsabilidades, bancos e comunicação de cada serviço
    E encontra instruções para executar testes e consultar Swagger/Postman

  Cenário: Vídeo demonstra requisitos dentro do limite
    Dado que o roteiro inclui o fluxo completo, compensação, deploy/testes e observabilidade
    Quando o vídeo final é revisado antes do envio
    Então sua duração é de no máximo 15 minutos
    E cada requisito pode ser confirmado por evidência visível

  Cenário: PDF está completo e acessível
    Dado que o PDF contém participantes, repositórios, vídeo, diagrama e decisões
    Quando o checklist do enunciado é aplicado
    Então nenhum campo obrigatório está ausente
    E os links abrem sem autorização interna do time
```

## Passos

1. Atualizar README por serviço a partir dos critérios executados, sem copiar texto genérico de outros repositórios.
2. Atualizar diagramas e coleção; validar referências cruzadas e links.
3. Capturar evidência por serviço de build, testes, cobertura, qualidade e deploy.
4. Gravar vídeo seguindo roteiro cronometrado; incluir falha e compensação.
5. Montar PDF único e revisar com checklist do enunciado.
6. Validar link do vídeo e acesso aos repos; submeter no portal e registrar confirmação.

## Evidências

- Links para os três repositórios e workflows, dashboards, diagramas, Swagger/Postman e relatório de testes.
- Roteiro e duração final do vídeo.
- PDF submetido e confirmação/registro do portal, sem publicar dados sensíveis.