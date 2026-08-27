# Diagrama de Entidade-Relacionamento

Banco único (RDS PostgreSQL) com dois schemas separados por bounded context, conforme [ADR-007](../adr/ADR-007-banco-compartilhado-schemas-separados.md). Não há foreign key entre schemas — `estoque.Pecas` é referenciada por `atendimento.ItensPeca.PecaId` apenas por convenção de ID, sem constraint, porque cada schema pertence a um serviço diferente (ver [ADR-004](../adr/ADR-004-estoque-fonte-verdade-pecas.md)).

```mermaid
erDiagram
    Cliente ||--o{ Veiculo : possui
    Cliente ||--o{ OrdemServico : solicita
    Veiculo ||--o{ OrdemServico : "é objeto de"
    OrdemServico ||--o{ ItemServico : contem
    OrdemServico ||--o{ ItemPeca : contem
    OrdemServico ||--o{ HistoricoStatusOS : registra
    Servico ||--o{ ItemServico : "é referenciado por"
    Peca ||--o{ ItemPeca : "é referenciado por (sem FK, outro schema)"

    Cliente {
        guid Id PK
        string Nome
        string Documento "CPF ou CNPJ, único"
        string Email
        string Telefone
        string Endereco
        bool Ativo
    }

    Veiculo {
        guid Id PK
        guid ClienteId FK
        string Placa "único"
        string Marca
        string Modelo
        int Ano
        string Cor
    }

    OrdemServico {
        guid Id PK
        string Numero "único, gerado sequencialmente"
        guid ClienteId FK
        guid VeiculoId FK
        enum Status "Recebida, EmDiagnostico, AguardandoAprovacao, EmExecucao, Finalizada, Entregue, Cancelada"
        string Observacoes
        datetime DataAbertura
        datetime DataFechamento "nullable"
    }

    ItemServico {
        guid Id PK
        guid OrdemServicoId FK
        guid ServicoId FK
        int Quantidade
        decimal ValorUnitario
    }

    ItemPeca {
        guid Id PK
        guid OrdemServicoId FK
        guid PecaId "referência lógica ao schema estoque, sem FK"
        int Quantidade
        decimal ValorUnitario
    }

    HistoricoStatusOS {
        guid Id PK
        guid OrdemServicoId FK
        enum StatusAnterior
        enum StatusNovo
        datetime DataAlteracao
    }

    Servico {
        guid Id PK
        string Nome
        string Descricao
        decimal Preco
        int TempoConclusaoMinutos
        bool Ativo
    }

    Usuario {
        guid Id PK
        string Nome
        string Email "único"
        string SenhaHash
        enum Perfil "Atendente, Mecanico, Admin"
        bool Ativo
    }

    Peca {
        guid Id PK
        string Nome
        string Descricao
        decimal Valor
        int QuantidadeEstoque
    }
```

## Schemas

| Schema | Tabelas | Serviço dono |
|---|---|---|
| `atendimento` | Cliente, Veiculo, OrdemServico, ItemServico, ItemPeca, HistoricoStatusOS, Servico, Usuario | Atendimento |
| `estoque` | Peca | Estoque |

## Relacionamentos e regras de negócio

- **Cliente → Veiculo**: um cliente pode ter vários veículos; um veículo pertence a exatamente um cliente. A abertura de uma OS valida que o veículo informado pertence ao cliente informado (`AbrirOrdemServicoUseCase`).
- **OrdemServico → ItemServico / ItemPeca**: uma OS agrega os serviços e peças usados no atendimento. `ValorTotal` da OS é calculado (não persistido) como a soma dos itens.
- **OrdemServico → HistoricoStatusOS**: cada transição de status gera um registro de histórico, incluindo status anterior e novo — auditoria completa do ciclo de vida da OS (ver [ADR-002](../adr/ADR-002-observabilidade-falhas-tabela-banco.md)).
- **ItemPeca → Peca (schema estoque)**: intencionalmente **sem foreign key** — `PecaId` é uma referência lógica resolvida via chamada HTTP síncrona (verificação de disponibilidade, na abertura) e evento assíncrono (baixa de estoque, na finalização), nunca via JOIN direto entre schemas. Essa é a fronteira do bounded context: o Atendimento não é dono do dado de peça, apenas referencia seu ID.
- **Usuario**: não tem relacionamento com as demais entidades — é o cadastro de login interno (atendentes/mecânicos/admin), independente do fluxo de Cliente (que se autentica via CPF, [RFC-003](../rfcs/RFC-003-estrategia-de-autenticacao.md)).

Ver também [Diagrama de Sequência — Abertura e Finalização de OS](diagrama-sequencia-abertura-os.md).
