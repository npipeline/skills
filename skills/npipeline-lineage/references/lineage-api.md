# Lineage API Reference

## Core Types

### LineagePacket<T>

When item-level lineage is enabled, streams carry `LineagePacket<T>` internally: the adapter wraps items at the source and unwraps them before each node, so node implementations still receive `T`. `LineagePacket<T>` is an implementation detail; you rarely construct it yourself.

```csharp
public sealed record LineagePacket<T>(
    T Data,
    Guid CorrelationId,
    ImmutableArray<string> TraversalPath) : ILineageEnvelope
{
    ImmutableArray<LineageRecord> LineageRecords { get; init; } = [];
    bool Collect { get; init; } = true;
}
```

> [!NOTE]
> `TraversalPath` and `LineageRecords` are `ImmutableArray<T>`, not `ImmutableList<T>`.

### LineageRecord

Lineage records each node's processing of an item:

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
| `ConsumedWithoutEmission` | Item was consumed by a sink or aggregate without producing output |
| `FilteredOut` | Item was filtered out |
| `DeadLettered` | Item was sent to dead-letter |
| `Error` | Item caused an error |
| `DroppedByBackpressure` | Item was dropped by queue backpressure |
| `Joined` | Item participated in a join |
| `Aggregated` | Item was aggregated into a group |

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
    LineageOverflowPolicy OverflowPolicy = LineageOverflowPolicy.Degrade,
    bool CaptureHopTimestamps = true,
    bool CaptureDecisions = true,
    bool CaptureObservedCardinality = true,
    bool CaptureAncestryMapping = false,
    bool CaptureHopSnapshots = false,
    int SampleEvery = 100,              // 1 = all items, 100 = 1%
    bool DeterministicSampling = true,  // Uses CorrelationId hash
    bool RedactData = true,             // Omit payload from records
    int MaxHopRecordsPerItem = 256,
    bool EnsurePerInputTerminalRecord = true,
    bool EmitBackpressureDropRecords = true,
    bool IncludeContributorCorrelationIds = true,
    bool EmitIntermediateNodeRecords = true,
    int AdapterBufferSize = 64)
```

### Sampling

By default, 1 in 100 items is tracked (`SampleEvery = 100`). Set to `1` for all items. When `DeterministicSampling = true`, the `CorrelationId` hash is used so the same items are tracked across runs.

### Preset Profiles

```csharp
LineageOptions.FastLineage       // 1/100 sampling, redacted, reduced detail; also the Default
LineageOptions.CompleteLineage   // 1/1 sampling, full data, ancestry, snapshots
```

`LineageOptions.ForProfile(LineageProfile.FastLineage)` / `ForProfile(LineageProfile.CompleteLineage)` select a preset by enum.

### Adapter Buffer

`AdapterBufferSize` (default 64) bounds how many items a transform's lineage adapter may read ahead of the transform when the node's lineage mapping streams. It keeps backpressure working with item-level lineage on. Values below 1 are treated as 1.

## Lineage Service

`LineageService` (the core implementation, implements `ILineage`) handles:

- Stream wrapping: wraps `IDataStream` with `LineagePacket<T>` at sources
- Stream unwrapping: strips lineage packets at sinks and before each node
- Adapter building: builds map/convert delegates for type transitions
- Cardinality mapping: several strategies for different cardinality patterns

### Cardinality Mapping Strategies

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
    Task RecordAsync(LineageRecord record, CancellationToken cancellationToken);
}
```

### IPipelineLineageSink (Pipeline-Level)

Receives the complete `PipelineLineageReport` after pipeline completion:

```csharp
public interface IPipelineLineageSink
{
    Task RecordAsync(PipelineLineageReport report, CancellationToken cancellationToken);
}
```

### Built-in: LoggingPipelineLineageSink

Serializes pipeline lineage as JSON to structured logs:

```csharp
builder.UseLoggingPipelineLineageSink();
// Or with a logger factory:
builder.UseLoggingPipelineLineageSink(loggerFactory);
```

`UseLoggingPipelineLineageSink()` registers the sink by type, so it logs through the container's logging (or the context's `ILoggerFactory` without DI).

## DI Registration

```csharp
// Basic: default sink
services.AddNPipelineLineage();

// Register lineage tracking without a default pipeline lineage sink
services.AddNPipelineLineageCore();

// With a custom pipeline-level lineage sink
services.AddNPipelineLineage<MyPipelineSink>();

// With a custom collector and sink
services.AddNPipelineLineage<MyCollector, MyPipelineSink>();
```

## Runner Without DI

A runner from `PipelineRunner.Create()` does not track lineage. Build the runner with `UseLineage()`:

```csharp
var runner = new PipelineRunnerBuilder().UseLineage().Build();
```

Without it, item-level lineage fails the build, and a configured pipeline lineage sink logs a warning instead of producing a report.

## Enabling in a Pipeline Definition

```csharp
public void Define(PipelineBuilder builder, PipelineContext context)
{
    builder.EnableItemLevelLineage(LineageOptions.FastLineage);

    // Or custom
    builder.EnableItemLevelLineage(opts => opts with { SampleEvery = 10, RedactData = false });
}
```

## Hop Snapshots

When `CaptureHopSnapshots = true`, lineage records include per-hop input/output data snapshots for debugging and Studio visualization. This is enabled in `CompleteLineage` mode. Performance impact is high; enable at conservative sampling rates.

## Cardinality Mismatch Detection

`LineageService` detects and reports cardinality mismatches (for example, a transform produces two outputs for one input without a declared mapper). Behavior is configured by:

- `Strict = true` → throws on mismatch
- `WarnOnMismatch = true` → logs a warning
- `OnMismatch` → custom callback
