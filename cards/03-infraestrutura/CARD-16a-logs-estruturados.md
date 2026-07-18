# CARD-16a — Logs estruturados

**Tipo:** Aplicação  
**Status:** To Do  
**Depende de:** nenhum  
**Bloqueia:** CARD-16b  
**Parte de:** [CARD-16](CARD-16-observabilidade.md)

---

## Contexto

Antes de instalar qualquer ferramenta de observabilidade, os logs das aplicações precisam estar em **JSON estruturado**. Sem isso:

- O Loki não consegue indexar campos individuais (`TraceId`, `OrdemServicoId`)
- O Grafana não consegue filtrar logs por atributo
- O `TraceId` propagado pelo OTel não aparece nos logs, tornando a correlação impossível

Log texto puro (ruim):
```
info: CriarOrdemServicoUseCase[0] OS criada para veículo abc-123
```

Log JSON estruturado (necessário):
```json
{
  "Timestamp": "2026-06-10T14:32:01Z",
  "Level": "Information",
  "Message": "OS criada com sucesso",
  "TraceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "SpanId": "00f067aa0ba902b7",
  "OrdemServicoId": "os-456",
  "VeiculoId": "abc-123"
}
```

---

## Critérios de aceite

- [ ] Saída de logs em JSON estruturado nos dois serviços (`AddJsonConsole` no `Program.cs`)
- [ ] `ILogger<T>` injetado e usado em todos os use cases de Atendimento
- [ ] `ILogger<T>` injetado e usado em todos os use cases e consumers de Estoque
- [ ] Eventos críticos cobertos com log (tabela abaixo)
- [ ] Erros logados com `LogError`, não `LogInformation`
- [ ] Nenhum dado sensível em nenhum log (senha, token JWT, connection string)

---

## Eventos obrigatórios

| Evento | Serviço | Nível | Campos obrigatórios |
|---|---|---|---|
| OS criada | Atendimento | `Information` | `OrdemServicoId`, `VeiculoId` |
| Status de OS alterado | Atendimento | `Information` | `OrdemServicoId`, `StatusAnterior`, `NovoStatus` |
| Orçamento aprovado/recusado | Atendimento | `Information` | `OrdemServicoId`, `Aprovado` |
| Evento publicado no RabbitMQ | Atendimento | `Information` | `EventoTipo`, `OrdemServicoId` |
| Evento consumido do RabbitMQ | Estoque | `Information` | `EventoTipo`, `OrdemServicoId` |
| Estoque baixado com sucesso | Estoque | `Information` | `PecaId`, `Quantidade`, `EstoqueRestante` |
| Falha ao processar evento | Estoque | `Error` | `OrdemServicoId`, `Motivo` |
| Falha de processamento registrada | Estoque | `Warning` | `OrdemServicoId`, `TentativaNumero` |

---

## Estrutura esperada no código

```csharp
// Program.cs — habilitar saída JSON
builder.Logging.AddJsonConsole(o =>
{
    o.IncludeScopes = true;         // inclui TraceId/SpanId quando OTel estiver ativo
    o.TimestampFormat = "O";        // ISO 8601
    o.JsonWriterOptions = new JsonWriterOptions { Indented = false };
});
```

```csharp
// Use case — estrutura correta
public class CriarOrdemServicoUseCase
{
    private readonly ILogger<CriarOrdemServicoUseCase> _logger;

    public async Task<OrdemServico> ExecutarAsync(CriarOrdemServicoCommand command)
    {
        _logger.LogInformation("Criando OS para veiculo {VeiculoId}", command.VeiculoId);

        // ... lógica ...

        _logger.LogInformation("OS {OrdemServicoId} criada com sucesso", os.Id);
        return os;
    }
}
```

---

## Passos

1. Verificar `Program.cs` dos dois serviços — formato de log atual
2. Adicionar `AddJsonConsole` se não estiver configurado
3. Auditar use cases do Atendimento: `ILogger<T>` presente? Eventos cobertos?
4. Auditar use cases e consumers do Estoque: idem
5. Corrigir pontos ausentes ou inadequados
6. Verificar ausência de dados sensíveis nos logs
7. Validar saída JSON rodando o serviço localmente e inspecionando `kubectl logs`
