# Metrics API Reference

## ObservabilitySurface

The central implementation (`IObservabilitySurface`) orchestrates pipeline and node lifecycle events:

```csharp
public interface IObservabilitySurface
{
    IPipelineActivity BeginPipeline<TDefinition>(PipelineContext context);
    Task CompletePipeline<TDefinition>(PipelineContext context, PipelineGraph graph, IPipelineActivity activity);
    Task FailPipeline<TDefinition>(PipelineContext context, Exception ex, IPipelineActivity activity);

    NodeObservationScope BeginNode(PipelineContext context, PipelineGraph graph, NodeDefinition nodeDef, INode nodeInstance);
    NodeExecutionCompleted CompleteNodeSuccess(PipelineContext context, NodeObservationScope scope);
    NodeExecutionCompleted CompleteNodeFailure(PipelineContext context, NodeObservationScope scope, Exception ex);
}
```

## ObservabilityCollector

Thread-safe collector using `ConcurrentDictionary`. Aggregates per-node metrics through a `NodeMetricsBuilder`:

```csharp
public interface IObservabilityCollector
{
    void RecordNodeStart(string nodeId, NodeMetrics metrics);
    void RecordNodeComplete(string nodeId, NodeExecutionCompleted completed);
    PipelineMetrics GetPipelineMetrics();
    void EmitToSink(IMetricsSink sink);
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

Throughput is calculated as `processed items / wall duration`.

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

Logs overall pipeline metrics:

```csharp
services.AddNPipelineObservability<LoggingMetricsSink, LoggingPipelineMetricsSink>();
```

Output:
```
Pipeline 'OrderPipeline' completed: 10000 items in 12.5s (800 items/s)
  5 nodes, 0 errors
```

## DI Registration

### AddNPipelineObservability

10 overloads with varying levels of customization. All return `IServiceCollection`:

```csharp
// Simplest — default sinks
services.AddNPipelineObservability();

// With typed sinks
services.AddNPipelineObservability<MyMetricsSink, MyPipelineSink>();

// With custom sinks and options
services.AddNPipelineObservability<MyMetricsSink, MyPipelineSink>(
    new ObservabilityExtensionOptions { EnableMemoryMetrics = true });

// With factory delegates
services.AddNPipelineObservability(
    sp => new MyMetricsSink(sp.GetRequiredService<ILogger<MyMetricsSink>>()),
    sp => new MyPipelineSink(sp.GetRequiredService<ILogger<MyPipelineSink>>()));

// With custom collector
services.AddNPipelineObservability<MyCollector, MyMetricsSink, MyPipelineSink>();

// With factory delegate for collector
services.AddNPipelineObservability<MyMetricsSink, MyPipelineSink>(
    sp => new MyCollector(sp));
```

## Enabling Observability on Nodes

```csharp
// In pipeline definition
source.WithObservability(builder);
transform.WithObservability(builder);
sink.WithObservability(builder);
join.WithObservability(builder);
aggregate.WithObservability(builder);
```

Extension methods available on `SourceNodeHandle`, `TransformNodeHandle`, `SinkNodeHandle`, `JoinNodeHandle`, and `AggregateNodeHandle`.

## PipelineMetrics

```csharp
public sealed record PipelineMetrics
{
    Guid PipelineId
    string PipelineName
    DateTimeOffset StartTime
    DateTimeOffset EndTime
    TimeSpan Duration
    int TotalNodes
    int SuccessfulNodes
    int FailedNodes
    long TotalItemsProcessed
    double OverallThroughput  // items/second
    IReadOnlyDictionary<string, NodeMetrics> NodeMetrics
}
```

## ExecutionObserver

`MetricsCollectingExecutionObserver` captures start/completion/retry events with memory and processor time deltas. Configured automatically when observability is enabled.
