---
name: npipeline-lineage
description: Use when the user wants to add data lineage tracking to NPipeline pipelines. Covers LineagePacket, item-level lineage, pipeline-level lineage, LineageService, sampling and redaction, cardinality mapping strategies, correlation trails, terminal outcomes, LoggingPipelineLineageSink, and DI setup with AddNPipelineLineage. Use when user mentions "data lineage", "provenance", "track data", "correlation", or "audit trail".
npipelineVersion: "0.52.0"
lastVerified: "2026-06-04"
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
| `FastLineage` (default) | 1/100 items | Data redacted | Reduced detail | Production throughput |
| `CompleteLineage` | 1/1 items | Data preserved | Full detail | Development, diagnostics, compliance |

```csharp
builder.EnableItemLevelLineage(LineageOptions.FastLineage);
builder.EnableItemLevelLineage(LineageOptions.CompleteLineage);
```

### Phase 2: Enable Lineage

```csharp
// In the pipeline definition
builder.EnableItemLevelLineage(opts => opts with { SampleEvery = 10 });
```

### Phase 3: Register Lineage Sinks (DI)

```csharp
// Pipeline-level lineage sink (most common)
services.AddNPipelineLineage<MyPipelineSink>();

// With custom collector
services.AddNPipelineLineage<MyCollector, MyPipelineSink>();

// Convenience: log lineage as JSON
builder.UseLoggingPipelineLineageSink();
```

### Phase 4: Inspect Lineage in Nodes

Nodes automatically receive `LineagePacket<T>` wrapped items when lineage is enabled. Access lineage metadata:

```csharp
public override async Task<EnrichedOrder> TransformAsync(
    LineagePacket<Order> item,  // Items are wrapped in LineagePacket
    PipelineContext context, CancellationToken ct)
{
    var correlationId = item.CorrelationId;
    var lineageRecords = item.LineageRecords;  // Per-node visit records
    var order = item.Data;      // The actual payload
    // ...
}
```

Consult `references/lineage-api.md` for the full LineageOptions, sink APIs, mapping strategies, and correlation details.
