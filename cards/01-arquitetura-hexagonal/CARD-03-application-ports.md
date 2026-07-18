# CARD-03 — Application: portas de saída (interfaces)

**Tipo:** Refactor  
**Status:** To Do  
**Depende de:** CARD-02  
**Bloqueia:** CARD-04, CARD-05

---

## Contexto

Na arquitetura hexagonal, o Application define as **portas de saída** — interfaces que descrevem o que ele precisa do mundo externo, sem saber como é implementado. Infrastructure vai implementar essas interfaces (CARD-05). Isso inverte a dependência: Infrastructure depende de Application, não o contrário.

---

## Critérios de aceite

- [ ] Todas as interfaces definidas no projeto `Application` (não no Domain)
- [ ] Nenhuma interface referencia tipos de EF Core, Npgsql ou qualquer lib de infra
- [ ] Stubs de `IEventPublisher`, `IEstoquePort` e `IEmailPort` documentados com comentário indicando que serão implementados em cards futuros
- [ ] `dotnet build` sem warnings

---

## Portas de saída a criar

### Repositórios
```
Application/Ports/Out/
├── IOrdemServicoRepository.cs
├── IClienteRepository.cs
├── IVeiculoRepository.cs
├── IUsuarioRepository.cs
├── IServicoRepository.cs
└── IPecaRepository.cs
```

### Integrações externas
```
Application/Ports/Out/
├── IEventPublisher.cs      // publica eventos no RabbitMQ — implementado no CARD-08
├── IEstoquePort.cs         // consulta disponibilidade de peças — implementado no CARD-09
└── IEmailPort.cs           // envia email de atualização de status — implementado no CARD-10
```

---

## Contratos relevantes

```csharp
// IEventPublisher.cs
public interface IEventPublisher
{
    Task PublishAsync<T>(string routingKey, T payload, CancellationToken ct = default);
}

// IEstoquePort.cs
public interface IEstoquePort
{
    Task<bool> VerificarDisponibilidadeAsync(IEnumerable<ItemPecaRequest> itens, CancellationToken ct = default);
}

// IEmailPort.cs
public interface IEmailPort
{
    Task EnviarAtualizacaoStatusAsync(string destinatario, string numeroOS, StatusOrdemServico novoStatus, CancellationToken ct = default);
}
```

---

## Passos

1. Criar pasta `Application/Ports/Out/`
2. Criar as interfaces de repositório (migrar e ajustar assinaturas do legado)
3. Criar as interfaces de integração com contratos acima
4. Adicionar comentários `// TODO: implementado em CARD-XX` nas integrações externas
5. Validar com `dotnet build`
