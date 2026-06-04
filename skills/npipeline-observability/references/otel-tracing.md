# OpenTelemetry Tracing Reference

## Package

```
dotnet add package NPipeline.Extensions.Observability.OpenTelemetry
```

## Core Types

### OpenTelemetryPipelineTracer

Wraps `System.Diagnostics.ActivitySource` with a service name:

```csharp
var tracer = new OpenTelemetryPipelineTracer("order-service");
// Creates activities scoped to "order-service" source
```

### PipelineActivity

Wraps `System.Diagnostics.Activity` for pipeline operations:

```csharp
public class PipelineActivity
{
    void SetTag(string key, string value);
    void SetTag(string key, double value);
    void RecordException(Exception ex);
}
```

## DI Registration

```csharp
// Simple — named service
services.AddOpenTelemetryPipelineTracer("order-service");

// Factory-based
services.AddOpenTelemetryPipelineTracer(sp =>
{
    var name = sp.GetRequiredService<IOptions<ServiceConfig>>().Value.Name;
    return new OpenTelemetryPipelineTracer(name);
});
```

## TracerProvider Configuration

Configure OpenTelemetry exporters via `TracerProviderBuilder` extensions:

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracerBuilder => tracerBuilder
        .AddNPipelineSource("order-service")
        .AddNPipelineSources("service-a", "service-b")
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("http://localhost:4317");
        }));
```

### Extension Methods

| Method | Purpose |
|---|---|
| `AddNPipelineSource(name)` | Add a single named NPipeline activity source |
| `AddNPipelineSources(name1, name2, ...)` | Add multiple named sources |
| `GetNPipelineInfo(activity)` | Extract pipeline metadata from activity tags |

## Supported Exporters

Since NPipeline uses standard `ActivitySource`, all OpenTelemetry exporters work:

| Exporter | NuGet Package |
|---|---|
| OTLP (gRPC/HTTP) | `OpenTelemetry.Exporter.OpenTelemetryProtocol` |
| Jaeger | `OpenTelemetry.Exporter.Jaeger` |
| Zipkin | `OpenTelemetry.Exporter.Zipkin` |
| Azure Monitor | `Azure.Monitor.OpenTelemetry.Exporter` |
| AWS X-Ray | `OpenTelemetry.Exporter.AWSXRay` |
| Console | `OpenTelemetry.Exporter.Console` |

## Trace Structure

Each pipeline run creates a root span with:
- Pipeline name and ID as tags
- Child spans for each node execution
- Exception recording on failure spans
- Duration recorded automatically via `Activity` timestamps

Node spans include:
- Node ID, node type, pipeline ID
- Start/end timestamps
- Items processed count (when metrics are enabled)

## Configuration

For development/debugging, use the console exporter:

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t
        .AddNPipelineSource("my-pipeline")
        .AddConsoleExporter());
```

## Example: Full Setup

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddNPipeline(typeof(Program).Assembly);
builder.Services.AddNPipelineObservability();

builder.Services.AddOpenTelemetryPipelineTracer("order-service");

builder.Services.AddOpenTelemetry()
    .WithTracing(t => t
        .AddNPipelineSource("order-service")
        .AddOtlpExporter(o => o.Endpoint = new Uri("http://jaeger:4317")));

var app = builder.Build();
```

This gives you:
- Per-node timing metrics logged via `LoggingMetricsSink`
- Distributed traces exported to Jaeger via OTLP
- Pipeline-level summary metrics
