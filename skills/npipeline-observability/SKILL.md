---
name: npipeline-observability
description: Use when the user wants to add observability to NPipeline pipelines. Covers metrics collection (processed items, durations, throughput), structured logging sinks, per-node timing breakdowns, execution observation, DI setup with AddNPipelineObservability, and OpenTelemetry distributed tracing (ActivitySource integration with Jaeger, Zipkin, Azure Monitor, OTLP exporters). Use when user mentions "metrics", "tracing", "OpenTelemetry", "monitor", "logs", "observe", or "telemetry".
npipelineVersion: "0.52.0"
lastVerified: "2026-06-04"
---

# NPipeline Observability

This skill covers adding observability to NPipeline pipelines: metrics collection, structured logging, execution observation, and OpenTelemetry distributed tracing.

## Packages

```
dotnet add package NPipeline.Extensions.Observability
dotnet add package NPipeline.Extensions.Observability.OpenTelemetry
```

## Workflow

When the user wants to add observability, determine what they need:

| Need | Package | What It Provides |
|---|---|---|
| Pipeline/node metrics (items, timing) | `Observability` | `ObservabilitySurface`, `ObservabilityCollector`, structured sinks |
| Distributed tracing (Jaeger, Zipkin, etc.) | `Observability.OpenTelemetry` | OpenTelemetry `ActivitySource` integration |
| Both | Both packages | Metrics + traces end-to-end |

### Phase 1: Add Metrics Collection

Register observability via DI:

```csharp
services.AddNPipelineObservability();
```

Enable metrics on individual nodes:

```csharp
// In pipeline definition
handle.WithObservability(builder);
```

This wraps the node with an `AutoObservabilityScope` that captures:
- **Work duration** — Time spent in the transform itself
- **Input wait duration** — Time waiting for input items
- **Output block duration** — Time blocked on output backpressure
- **Wall duration** — Total wall-clock time
- **Throughput** — Items processed per second

### Phase 2: Choose a Metrics Sink

```csharp
// Logging sink (structured logs) — per-node metrics
services.AddNPipelineObservability<LoggingMetricsSink, LoggingPipelineMetricsSink>();

// Custom sinks — specify both per-node and pipeline-level sinks
services.AddNPipelineObservability<MyMetricsSink, MyPipelineSink>();

// With factory delegate
services.AddNPipelineObservability(
    sp => new MyMetricsSink(sp.GetRequiredService<ILogger<MyMetricsSink>>()),
    sp => new MyPipelineSink(sp.GetRequiredService<ILogger<MyPipelineSink>>()));
```

Built-in sinks:
- `LoggingMetricsSink` — Logs per-node metrics as structured log events
- `LoggingPipelineMetricsSink` — Logs overall pipeline metrics (total items, duration, throughput)

### Phase 3: Add OpenTelemetry Tracing (optional)

```csharp
services.AddOpenTelemetryPipelineTracer("order-service");

// Configure exporter
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t
        .AddNPipelineSource("order-service")
        .AddOtlpExporter());
```

Supported exporters: Jaeger, Zipkin, Azure Monitor, AWS X-Ray, OTLP (any OTLP-compatible backend).

See `references/metrics-api.md` for detailed metrics and sinks API.
See `references/otel-tracing.md` for OpenTelemetry integration and exporter configuration.
