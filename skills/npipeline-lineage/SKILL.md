---
name: npipeline-lineage
description: Use when the user wants to add data lineage tracking to NPipeline pipelines. Covers LineagePacket, item-level lineage, pipeline-level lineage, LineageService, sampling and redaction, cardinality mapping strategies, correlation trails, terminal outcomes, LoggingPipelineLineageSink, and DI setup with AddNPipelineLineage. Use when user mentions "data lineage", "provenance", "track data", "correlation", or "audit trail".
npipelineVersion: "0.67.0"
lastVerified: "2026-09-29"
---

# NPipeline Lineage

This skill covers `NPipeline.Extensions.Lineage` — data provenance tracking that records the journey of each item through the pipeline graph, including correlation IDs, hop records, terminal outcomes, and cardinality analysis.

## Package

```
dotnet add package NPipeline.Extensions.Lineage
```

## Workflow

When the user wants data lineage, determine the depth of tracking they need, then configure appropriately.

### Phase 1: Choose a Lineage Profile

Two presets:

| Profile | Sampling | Redaction | Detail | Use Case |
|---|---|---|---|---|
| `FastLineage` (the `LineageOptions.Default`) | 1/100 items | Data redacted | Reduced detail | Production throughput |
| `CompleteLineage` | 1/1 items | Data preserved | Full detail | Development, diagnostics, compliance |

```csharp
builder.EnableItemLevelLineage(LineageOptions.FastLineage);
builder.EnableItemLevelLineage(LineageOptions.CompleteLineage);
```

### Phase 2: Enable Lineage

```csharp
// In the pipeline definition. With no argument, CompleteLineage is used.
builder.EnableItemLevelLineage();

// Or customize
builder.EnableItemLevelLineage(opts => opts with { SampleEvery = 10 });
```

> [!IMPORTANT]
> Item-level lineage requires the `NPipeline.Extensions.Lineage` package. If the builder was created without a lineage module, `Build()` throws. With DI, call `services.AddNPipelineLineage()`. Without DI, build the runner with `new PipelineRunnerBuilder().UseLineage()`; a runner from `PipelineRunner.Create()` does not track lineage.

### Phase 3: Register Lineage Sinks (DI)

```csharp
// Pipeline-level lineage sink (most common)
services.AddNPipelineLineage<MyPipelineSink>();

// With a custom collector and sink
services.AddNPipelineLineage<MyCollector, MyPipelineSink>();

// Convenience: log pipeline lineage as JSON
builder.UseLoggingPipelineLineageSink();
```

### Phase 4: Inspect Lineage

Node implementations receive the plain item type (`T`), **not** `LineagePacket<T>`. The lineage adapter wraps items at the source and unwraps them before each node:

```csharp
public override async ValueTask<EnrichedOrder> TransformAsync(
    Order order, PipelineContext context, CancellationToken ct)
{
    // `order` is the payload; lineage is tracked around this call.
    return await EnrichAsync(order, ct);
}
```

Item payload snapshots and records are captured by the framework, not read off the item in node code. To consume lineage programmatically, implement an `ILineageSink` / `IPipelineLineageSink` (Phase 3) or query the `ILineageCollector`.

Consult `references/lineage-api.md` for the full `LineageOptions`, sink APIs, mapping strategies, and correlation details.
