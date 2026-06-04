# Lineage API Reference

## Core Types

### LineagePacket<T>

Items are wrapped in `LineagePacket<T>` when item-level lineage is enabled:

```csharp
public sealed record LineagePacket<T>(
    T Data,
    Guid CorrelationId,
    ImmutableList<string> TraversalPath) : ILineageEnvelope
{
    ImmutableList<LineageRecord> LineageRecords { get; init; } = ImmutableList<LineageRecord>.Empty;
    bool Collect { get; init; } = true;
}
```

### LineageRecord

The lineage system uses `LineageRecord` (not a separate `HopRecord` type) to record each node's processing of an item:

```csharp
public sealed record LineageRecord(
    Guid CorrelationId,
    string NodeId,
    Guid PipelineId,
    LineageOutcomeReason OutcomeReason,
    bool IsTerminal,
    IReadOnlyList<string> TraversalPath,
    string? PipelineName = null,
    DateTimeOffset TimestampUtc = default,
    int? RetryCount = null,
    IReadOnlyList<Guid>? ContributorCorrelationIds = null,
    IReadOnlyList<int>? ContributorInputIndices = null,
    int? InputContributorCount = null,
    int? OutputEmissionCount = null,
    ObservedCardinality Cardinality = ObservedCardinality.Unknown,
    object? InputSnapshot = null,
    object? OutputSnapshot = null,
    object? Data = null)
```

### LineageOutcomeReason

| Outcome | Meaning |
|---|---|
| `Emitted` | Item passed through normally |
| `ConsumedWithoutEmission` | Item was consumed by a sink without producing output |
| `FilteredOut` | Item was filtered out |
| `DeadLettered` | Item was sent to dead-letter |
| `Error` | Item caused an error |
| `DroppedByBackpressure` | Item was dropped by queue backpressure |
| `Joined` | Item participated in a join |
| `Aggregated` | Item was aggregated into a group |

### CorrelationTrail

The `LineageCollector` maintains a timeline of `LineageRecord` entries per correlation ID — a `CorrelationTrail` shows the item's path through the pipeline.

### PipelineLineageReport

Aggregated view of all lineage records from one pipeline run:

```csharp
public sealed record PipelineLineageReport(
    string Pipeline,
    Guid RunId,
    IReadOnlyList<NodeLineageInfo> Nodes,
    IReadOnlyList<EdgeLineageInfo> Edges,
    Guid PipelineId)
```

## LineageOptions

```csharp
public sealed record LineageOptions(
    bool Strict = false,
    bool WarnOnMismatch = true,
    Action<LineageMismatchContext>? OnMismatch = null,
    int? MaterializationCap = null,
    LineageOverflowPolicy OverflowPolicy = Degrade,
    bool CaptureHopTimestamps = true,
    bool CaptureDecisions = true,
    bool CaptureObservedCardinality = true,
    bool CaptureAncestryMapping = false,
    bool CaptureHopSnapshots = false,
    int SampleEvery = 100,              // 1 = all items, 100 = 1%
    bool DeterministicSampling = true,   // Uses CorrelationId hash
    bool RedactData = true,             // Omit payload from records
    int MaxHopRecordsPerItem = 256,
    bool EnsurePerInputTerminalRecord = true,
    bool EmitBackpressureDropRecords = true,
    bool IncludeContributorCorrelationIds = true,
    bool EmitIntermediateNodeRecords = true)
```

### Sampling

By default, 1 in 100 items is tracked (`SampleEvery = 100`). Set to `1` for all items. When `DeterministicSampling = true`, the CorrelationId hash is used so the same items are tracked across runs.

### Preset Profiles

```csharp
LineageOptions.FastLineage       // 1/100 sampling, redacted, reduced detail
LineageOptions.CompleteLineage   // 1/1 sampling, full data, ancestry, snapshots
```

## Lineage Service

`LineageService` (the core implementation, implements `ILineage`) handles:

- Stream wrapping — wraps `IDataStream` with `LineagePacket<T>` at sources
- Stream unwrapping — strips lineage packets at sinks
- Compiled expression tree-based type-aware wrappers (avoids reflection)
- Adapter building — builds map/convert delegates for type transitions
- Cardinality mapping — 4 strategies for different cardinality patterns

## Cardinality Mapping Strategies

When lineage passes through a node with different input/output cardinality:

| Strategy | Cardinality | Behavior |
|---|---|---|
| `StreamingOneToOneStrategy` | 1:1 | Maps each input directly to output (no materialization) |
| `MaterializingStrategy` | N:1 or 1:N | Materializes all inputs, maps to output(s) |
| `PositionalStreamingStrategy` | N:M (positional) | Maps by position without full materialization |
| `CapAwareMaterializingStrategy` | N:M (capped) | Materializes up to `MaterializationCap`, then degrades |

## Sinks

### ILineageSink (Item-Level)

Receives individual `LineageRecord` items:

```csharp
public interface ILineageSink
{
    Task HandleAsync(LineageRecord record, CancellationToken ct);
}
```

### IPipelineLineageSink (Pipeline-Level)

Receives the complete `PipelineLineageReport` after pipeline completion:

```csharp
public interface IPipelineLineageSink
{
    Task HandleAsync(PipelineLineageReport report, CancellationToken ct);
}
```

### Built-in: LoggingPipelineLineageSink

Serializes pipeline lineage as JSON to structured logs:

```csharp
builder.UseLoggingPipelineLineageSink();
```

## DI Registration

```csharp
// Basic — use defaults
services.AddNPipelineLineage();

// With custom pipeline-level lineage sink
services.AddNPipelineLineage<MyPipelineSink>();

// With custom lineage sink
services.AddNPipelineLineage<MyPipelineSink>(sp => new MyCollector());

// With custom collector and sink
services.AddNPipelineLineage<MyCollector, MyPipelineSink>();

// Convenience: log lineage as JSON
builder.UseLoggingPipelineLineageSink();
```

## Enabling in Pipeline Definition

```csharp
public void Define(PipelineBuilder builder, PipelineContext context)
{
    builder.EnableItemLevelLineage(LineageOptions.FastLineage);

    // Or custom
    builder.EnableItemLevelLineage(opts => opts
        .With(sampleEvery: 10)
        .With(redactData: false));
}
```

## Hop Snapshots

When `CaptureHopSnapshots = true`, lineage records include per-hop input/output data snapshots for debugging and Studio visualization. This is enabled in `CompleteLineage` mode.

## Cardinality Mismatch Detection

The `LineageService` detects and reports cardinality mismatches (e.g., a transform produces 0 outputs for an input, or 2 outputs for 1 input without a declared mapper). Behavior configured by:

- `Strict = true` → throws on mismatch
- `WarnOnMismatch = true` → logs warning
- `OnMismatch` → custom callback
