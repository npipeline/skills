# Metrics API Reference

## ObservabilitySurface

The central implementation (`IObservabilitySurface`) orchestrates pipeline and node lifecycle events:

```csharp
public interface IObservabilitySurface
{
    IPipelineActivity BeginPipeline<TDefinition>(PipelineContext context) where TDefinition : IPipelineDefinition, new();
    Task CompletePipeline<TDefinition>(PipelineContext context, PipelineGraph graph, IPipelineActivity activity);
    Task FailPipeline<TDefinition>(PipelineContext context, Exception ex, IPipelineActivity activity);

    NodeObservationScope BeginNode(PipelineContext context, PipelineGraph graph, NodeDefinition nodeDef, INode nodeInstance);
    NodeExecutionCompleted CompleteNodeSuccess(PipelineContext context, NodeObservationScope scope);
    NodeExecutionCompleted CompleteNodeFailure(PipelineContext context, NodeObservationScope scope, Exception ex);
}
```

## IObservabilityCollector

The collector is keyed by both node id and pipeline id (a run can nest sub-pipelines):

```csharp
public interface IObservabilityCollector
{
    void RecordNodeStart(string nodeId, DateTimeOffset timestamp, Guid pipelineId, int? threadId = null,
        double? initialMemoryMb = null, string? pipelineName = null);
    void RecordNodeEnd(string nodeId, DateTimeOffset timestamp, bool success, Guid pipelineId,
        Exception? exception = null, double? peakMemoryMb = null, double? processorTimeMs = null, string? pipelineName = null);
    void RecordItemMetrics(string nodeId, long itemsProcessed, long itemsEmitted, Guid pipelineId, string? pipelineName = null);
    void RecordNodeKind(string nodeId, NodeKind kind, Guid pipelineId, string? pipelineName = null);       // default impl
    void RecordItemsReplayed(string nodeId, long itemsReplayed, Guid pipelineId, string? pipelineName = null); // default impl
    void RecordRetry(string nodeId, int retryCount, Guid pipelineId, string? reason = null, string? pipelineName = null);
    void RecordRetryExhausted(string nodeId, Guid pipelineId, string? pipelineName = null);                 // default impl
    void RecordCircuitStateChanged(string nodeId, CircuitState state, Guid pipelineId, string? pipelineName = null);
    void RecordPerformanceMetrics(string nodeId, double throughputItemsPerSec, double averageItemProcessingMs,
        Guid pipelineId, string? pipelineName = null);
    void RecordTimingBreakdown(string nodeId, NodeTimingBreakdown timingBreakdown, Guid pipelineId, string? pipelineName = null);

    IReadOnlyList<INodeMetrics> GetNodeMetrics();
    INodeMetrics? GetNodeMetrics(string nodeId, Guid pipelineId);
    void ReleasePipeline(Guid pipelineId);   // default impl; releases a run's metrics
    IPipelineMetrics CreatePipelineMetrics(string pipelineName, Guid pipelineId, Guid runId, DateTimeOffset startTime,
        DateTimeOffset? endTime, bool success, Exception? exception = null);
    Task EmitMetricsAsync(string pipelineName, Guid pipelineId, Guid runId, DateTimeOffset startTime,
        DateTimeOffset? endTime, bool success, Exception? exception = null, CancellationToken cancellationToken = default);
}
```

## NodeTimingBreakdown

Per-node timing captures four distinct buckets:

| Metric | Description |
|---|---|
| `WorkDuration` | Time spent inside the transform logic |
| `InputWaitDuration` | Time the node waited for upstream to produce items |
| `OutputBlockDuration` | Time the node was blocked by downstream backpressure |
| `WallDuration` | Total wall-clock time (work + wait + block) |

Throughput is `processed items / wall duration`.

## Metrics Sinks

### LoggingMetricsSink

Logs per-node metrics at structured level:

```csharp
services.AddNPipelineObservability<LoggingMetricsSink, LoggingPipelineMetricsSink>();
```

Output (each node, after completion):
```
Node 'validate' completed: 1000 items in 2.3s (434 items/s)
  Work: 1.8s, Input wait: 0.3s, Output block: 0.2s
```

### LoggingPipelineMetricsSink

Logs pipeline items in and out, duration, and throughput:

```csharp
services.AddNPipelineObservability<LoggingMetricsSink, LoggingPipelineMetricsSink>();
```

Output:
```
Pipeline 'OrderPipeline' completed: 10000 items in 12.5s (800 items/s)
  5 nodes, 0 errors
```

A node without observability options reports `ItemCountsRecorded = false`, and the sinks say its item counts were not recorded instead of logging "Processed 0 items".

## DI Registration

### AddNPipelineObservability

Overloads with varying levels of customization. All return `IServiceCollection`:

```csharp
// Simplest: default options
services.AddNPipelineObservability();

// With options
services.AddNPipelineObservability(new ObservabilityExtensionOptions { AutoObserveAllNodes = true });

// With typed sinks
services.AddNPipelineObservability<MyMetricsSink, MyPipelineSink>();

// With typed sinks and options
services.AddNPipelineObservability<MyMetricsSink, MyPipelineSink>(options);

// With factory delegates
services.AddNPipelineObservability(
    sp => new MyMetricsSink(sp.GetRequiredService<ILogger<MyMetricsSink>>()),
    sp => new MyPipelineSink(sp.GetRequiredService<ILogger<MyPipelineSink>>()));

// With a custom collector
services.AddNPipelineObservability<MyCollector, MyMetricsSink, MyPipelineSink>();
```

`AddNPipelineObservability` is safe to call more than once: the first call that passes options sets them, and it adds its metrics observer alongside (not instead of) an observer the app registered.

### ConfigureNPipelineObservability

Adjusts the options whether it is called before or after `AddNPipelineObservability`:

```csharp
services.ConfigureNPipelineObservability(o => o with { AutoObserveAllNodes = true });
```

## Enabling Observability on Nodes

```csharp
// In pipeline definition
source.WithObservability(builder);
transform.WithObservability(builder);
sink.WithObservability(builder);
join.WithObservability(builder);
aggregate.WithObservability(builder);

// With explicit options
transform.WithObservability(builder, ObservabilityOptions.Full);
```

Extension methods are available on `SourceNodeHandle`, `TransformNodeHandle`, `SinkNodeHandle`, `AggregateNodeHandle`, and `JoinNodeHandle`.

## PipelineMetrics

```csharp
public sealed record PipelineMetrics(
    string PipelineName,
    Guid PipelineId,
    Guid RunId,
    DateTimeOffset StartTime,
    DateTimeOffset? EndTime,
    double? DurationMs,
    bool Success,
    long TotalItemsProcessed,
    IReadOnlyList<INodeMetrics> NodeMetrics,
    Exception? Exception,
    long? ItemsIn = null,      // items the pipeline's source nodes emitted
    long? ItemsOut = null) : IPipelineMetrics;   // items the pipeline's sink nodes processed
```

`ItemsIn` and `ItemsOut` are `null` when no node recorded item counts. `TotalItemsProcessed` is the sum across nodes and includes sources, sinks, joins, and aggregates.

`INodeMetrics.Kind` reports the node's kind. `NodeMetrics` also carries `ItemsReplayed` and `ItemCountsRecorded`.

## ExecutionObserver

`MetricsCollectingExecutionObserver` captures start, completion, retry, circuit-state, and exhausted-retry events with memory and processor-time deltas. It is registered automatically by `AddNPipelineObservability`.
