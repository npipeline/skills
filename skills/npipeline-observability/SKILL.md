---
name: npipeline-observability
description: Use when the user wants to add observability to NPipeline pipelines. Covers metrics collection (items in/out, per-node item counts, durations, throughput), structured logging sinks, per-node timing breakdowns, execution observation, DI setup with AddNPipelineObservability and ConfigureNPipelineObservability, AutoObserveAllNodes, and OpenTelemetry distributed tracing (ActivitySource integration with Jaeger, Zipkin, Azure Monitor, OTLP exporters). Use when user mentions "metrics", "tracing", "OpenTelemetry", "monitor", "logs", "observe", or "telemetry".
npipelineVersion: "0.67.0"
lastVerified: "2026-09-29"
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
// In the pipeline definition
handle.WithObservability(builder);
```

This wraps the node with an `AutoObservabilityScope` that captures work duration, input-wait duration, output-block duration, wall duration, throughput, and item counts.

> [!IMPORTANT]
> A run under `AddNPipelineObservability` must use a context wired to the container, created with `serviceProvider.CreatePipelineContext(...)` (or `RunPipelineAsync`). A run with `new PipelineContext()` or `PipelineContext.CreateDefault()` has no collector and records nothing; it logs a warning. The same warning is logged for a runner from `PipelineRunner.Create()` whose nodes use `WithObservability`.

### Phase 2: Choose a Metrics Sink

```csharp
// Logging sinks (structured logs) — per-node and pipeline-level
services.AddNPipelineObservability<LoggingMetricsSink, LoggingPipelineMetricsSink>();

// Custom sinks
services.AddNPipelineObservability<MyMetricsSink, MyPipelineSink>();

// With factory delegates
services.AddNPipelineObservability(
    sp => new MyMetricsSink(sp.GetRequiredService<ILogger<MyMetricsSink>>()),
    sp => new MyPipelineSink(sp.GetRequiredService<ILogger<MyPipelineSink>>()));
```

Built-in sinks:
- `LoggingMetricsSink` — logs per-node metrics as structured log events
- `LoggingPipelineMetricsSink` — logs pipeline items in and out, duration, and throughput

To adjust options whether `AddNPipelineObservability` is called before or after:

```csharp
services.ConfigureNPipelineObservability(o => o with { AutoObserveAllNodes = true });
```

`ObservabilityExtensionOptions.AutoObserveAllNodes` (default `false`) observes nodes that were not configured with `WithObservability`. A node's own options take precedence.

### Phase 3: Add OpenTelemetry Tracing (optional)

```csharp
services.AddOpenTelemetryPipelineTracer("order-service");

builder.Services.AddOpenTelemetry()
    .WithTracing(t => t
        .AddNPipelineSource("order-service")
        .AddOtlpExporter());
```

Supported exporters: Jaeger, Zipkin, Azure Monitor, AWS X-Ray, OTLP (any OTLP-compatible backend).

See `references/metrics-api.md` for the metrics and sinks API.
See `references/otel-tracing.md` for OpenTelemetry integration and exporter configuration.
