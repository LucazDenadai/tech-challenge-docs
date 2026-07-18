# CARD-16b — Instrumentação OpenTelemetry no código

**Tipo:** Aplicação  
**Status:** To Do  
**Depende de:** CARD-16a (logs estruturados)  
**Bloqueia:** CARD-16c  
**Parte de:** [CARD-16](CARD-16-observabilidade.md)

---

## Contexto

OpenTelemetry é o SDK que instrumenta o código para capturar traces e métricas automaticamente. Configurado uma vez no `Program.cs`, ele intercepta todas as requisições HTTP, queries SQL e mensagens de fila sem precisar alterar cada endpoint.

**O que o OTel captura automaticamente:**
- Cada `POST /ordens-servico` → span com duração, status HTTP, rota
- Cada query SQL executada pelo EF Core → span filho com o SQL executado
- Cada `Publish` e `Consume` do MassTransit → span com nome do evento e broker
- O `TraceId` é propagado via headers HTTP e headers de mensagem do RabbitMQ — o span do Estoque fica filho do span do Atendimento automaticamente

---

## Critérios de aceite

- [ ] Pacotes NuGet de OTel instalados em `Atendimento.API` e `Estoque.API`
- [ ] `OpenTelemetryExtensions.cs` criado nos dois projetos
- [ ] Traces exportados para Jaeger via OTLP (gRPC porta 4317)
- [ ] Métricas exportadas via endpoint `/metrics` (Prometheus scrape)
- [ ] `Jaeger__Endpoint` configurado nos `appsettings.json` e ConfigMaps K8s
- [ ] `TraceId` aparece nos logs (automático quando OTel + `AddJsonConsole` estão ativos juntos)

---

## Pacotes NuGet (por projeto de API)

```xml
<PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.9.0" />
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.9.0" />
<PackageReference Include="OpenTelemetry.Instrumentation.EntityFrameworkCore" Version="1.0.0-beta.12" />
<PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.9.0" />
<PackageReference Include="OpenTelemetry.Exporter.Prometheus.AspNetCore" Version="1.9.0-rc.1" />
<PackageReference Include="OpenTelemetry.Instrumentation.MassTransit" Version="1.0.0-beta.3" />
```

> **Por que OTLP e não Jaeger direto?**  
> O exporter OTLP (OpenTelemetry Protocol) é o padrão moderno — envia para qualquer backend compatível (Jaeger, Tempo, Zipkin) sem trocar o SDK. O Jaeger aceita OTLP nativamente desde a versão 1.35.

---

## Código — `OpenTelemetryExtensions.cs`

```csharp
public static class OpenTelemetryExtensions
{
    public static IHostApplicationBuilder AddOpenTelemetry(
        this IHostApplicationBuilder builder, string serviceName)
    {
        var jaegerEndpoint = builder.Configuration["Jaeger__Endpoint"]
            ?? "http://jaeger:4317";

        builder.Services.AddOpenTelemetry()
            .WithTracing(tracing => tracing
                .SetResourceBuilder(ResourceBuilder.CreateDefault()
                    .AddService(serviceName))
                .AddAspNetCoreInstrumentation(o =>
                {
                    o.RecordException = true;
                })
                .AddEntityFrameworkCoreInstrumentation(o =>
                {
                    o.SetDbStatementForText = true;
                })
                .AddMassTransitInstrumentation()
                .AddOtlpExporter(o =>
                {
                    o.Endpoint = new Uri(jaegerEndpoint);
                    o.Protocol = OtlpExportProtocol.Grpc;
                }))
            .WithMetrics(metrics => metrics
                .SetResourceBuilder(ResourceBuilder.CreateDefault()
                    .AddService(serviceName))
                .AddAspNetCoreInstrumentation()
                .AddRuntimeInstrumentation()
                .AddPrometheusExporter());

        return builder;
    }
}
```

## Registro no `Program.cs`

```csharp
// Atendimento
builder.AddOpenTelemetry("OficinaMecanica.Atendimento");

// ...após builder.Build()...
app.MapPrometheusScrapingEndpoint(); // expõe /metrics
```

---

## Configuração — `appsettings.json`

```json
{
  "Jaeger__Endpoint": "http://localhost:4317"
}
```

## Configuração — ConfigMap K8s (`atendimento/configmap.yaml`)

Adicionar ao ConfigMap existente:

```yaml
Jaeger__Endpoint: http://jaeger-svc:4317
```

---

## Passos

1. Instalar pacotes NuGet nos dois projetos de API
2. Criar `src/Atendimento/.../Extensions/OpenTelemetryExtensions.cs`
3. Criar `src/Estoque/.../Extensions/OpenTelemetryExtensions.cs`
4. Registrar nos dois `Program.cs` e adicionar `MapPrometheusScrapingEndpoint()`
5. Adicionar `Jaeger__Endpoint` nos `appsettings.json` de desenvolvimento
6. Adicionar `Jaeger__Endpoint` nos ConfigMaps K8s de Atendimento e Estoque
7. Build e verificar que `dotnet build` passa com 0 erros
