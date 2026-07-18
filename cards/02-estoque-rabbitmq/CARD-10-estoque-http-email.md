# CARD-10 — Adapters reais: EstoqueHttp + Email

**Tipo:** Feature  
**Status:** To Do  
**Depende de:** CARD-08, CARD-09  
**Bloqueia:** nenhum neste épico

---

## Contexto

Substituir os dois stubs restantes do Atendimento:
1. `EstoqueHttpStub` → chamada HTTP real ao serviço de Estoque para verificar disponibilidade na abertura de OS
2. `EmailStub` → envio real de email ao cliente quando o status da OS é atualizado

---

## Critérios de aceite

- [ ] `EstoqueHttpAdapter` chama `POST /estoque/disponibilidade` com Polly (circuit breaker + retry)
- [ ] Se Estoque indisponível, abertura de OS retorna `503` com mensagem clara
- [ ] Email enviado via SMTP ao atualizar status da OS
- [ ] Credenciais SMTP em variável de ambiente (não hardcoded)
- [ ] `dotnet test` passa com mocks dos adapters

---

## EstoqueHttpAdapter

```
Atendimento/Infrastructure/Adapters/Out/Http/
└── EstoqueHttpAdapter.cs    // implementa IEstoquePort
```

Polly configurado com:
- Retry: 2 tentativas com backoff de 500ms
- Circuit breaker: abre após 3 falhas consecutivas, fecha após 30s

```csharp
builder.Services.AddHttpClient<IEstoquePort, EstoqueHttpAdapter>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["EstoqueServiceUrl"]!);
})
.AddPolicyHandler(HttpPolicyExtensions
    .HandleTransientHttpError()
    .WaitAndRetryAsync(2, _ => TimeSpan.FromMilliseconds(500)))
.AddPolicyHandler(HttpPolicyExtensions
    .HandleTransientHttpError()
    .CircuitBreakerAsync(3, TimeSpan.FromSeconds(30)));
```

---

## EmailSmtpAdapter

```
Atendimento/Infrastructure/Adapters/Out/Email/
└── EmailSmtpAdapter.cs    // implementa IEmailPort
```

Variáveis de ambiente necessárias:
```
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=oficina@exemplo.com
SMTP_PASSWORD=<secret>
SMTP_FROM=Oficina Mecânica <oficina@exemplo.com>
```

Template do email:
```
Assunto: [Oficina] OS {numeroOS} — Status atualizado

Olá {nomeCliente},

Sua ordem de serviço {numeroOS} teve o status atualizado para:
{novoStatus}

Acesse nosso sistema para mais detalhes.
```

---

## Passos

1. Implementar `EstoqueHttpAdapter` com Polly
2. Registrar `IEstoquePort` → `EstoqueHttpAdapter` no DI do Atendimento
3. Adicionar `EstoqueServiceUrl` ao `appsettings.json` e `docker-compose.yml`
4. Implementar `EmailSmtpAdapter` com MailKit ou System.Net.Mail
5. Registrar `IEmailPort` → `EmailSmtpAdapter` no DI
6. Adicionar variáveis SMTP ao `.env.example` e Secrets K8s (CARD-13)
7. Validar com teste manual: finalizar OS e checar email recebido
