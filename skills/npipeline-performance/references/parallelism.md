# Parallel Execution Reference

## Package

```
dotnet add package NPipeline.Extensions.Parallelism
```

## Parallel Execution Strategies

Three strategies built on `System.Threading.Channels` with configurable backpressure:

| Strategy | Backpressure | Use Case |
|---|---|---|
| `BlockingParallelStrategy` | Blocks producer when queue full | General-purpose, preserve ordering |
| `DropOldestParallelStrategy` | Drops oldest items when queue full | Real-time streaming (new > old) |
| `DropNewestParallelStrategy` | Drops newest items when queue full | Back-pressure sensitivity (old > new) |

## ParallelOptions

```csharp
public sealed record ParallelOptions(
    int? MaxDegreeOfParallelism = null,       // null = Environment.ProcessorCount
    int? MaxQueueLength = null,               // null = bounded only by upstream
    BoundedQueuePolicy QueuePolicy = BoundedQueuePolicy.Block,
    int? OutputBufferCapacity = null,
    bool PreserveOrdering = true,
    TimeSpan? MetricsInterval = null,         // default 1 second
    bool EnableInputWaitTiming = false);
```

### Workload Presets

```csharp
handle.RunParallel(builder, ParallelWorkloadType.CpuBound);
handle.RunParallel(builder, ParallelWorkloadType.IoBound);
```

| Preset | MaxDoP | Max Queue | Queue Policy | Best For |
|---|---|---|---|---|
| `General` | CPU × 2 | CPU × 4 | Block | Default catch-all |
| `CpuBound` | CPU | CPU × 2 | Block | CPU-heavy transforms |
| `IoBound` | CPU × 4 | CPU × 8 | Block | Database, HTTP, file I/O |
| `NetworkBound` | min(CPU × 8, 100) | 200 | Block | High-latency network calls |

### Custom Configuration

```csharp
handle.WithBlockingParallelism(builder,
    maxDegreeOfParallelism: 8,
    maxQueueLength: 2048,
    outputBufferCapacity: 256);

handle.WithUnorderedParallelism(builder,
    maxDegreeOfParallelism: 8,
    maxQueueLength: 2048);

handle.WithDropOldestParallelism(builder,
    maxDegreeOfParallelism: 16,
    maxQueueLength: 4096);

handle.WithDropNewestParallelism(builder,
    maxDegreeOfParallelism: 16,
    maxQueueLength: 4096);
```

### Advanced: Full Custom

```csharp
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = 8,
    MaxQueueLength = 2048,
    QueuePolicy = BoundedQueuePolicy.Block,
    OutputBufferCapacity = 256,
    PreserveOrdering = true,
    MetricsInterval = TimeSpan.FromSeconds(30)
};

handle.RunParallel(builder, opt => opt
    .MaxDegreeOfParallelism(8)
    .MaxQueueLength(50)
    .BlockOnBackpressure()
    .OutputBufferCapacity(200)
    .AllowUnorderedOutput());
```

`ParallelOptionsBuilder` also provides `DropOldestOnBackpressure()`, `DropNewestOnBackpressure()`, `EnableInputWaitTiming()`, and `MetricsInterval(TimeSpan)`.

## Parallel Execution Metrics

When metrics are enabled (via `MetricsInterval`), the strategy tracks:

- `Processed` — items processed
- `Enqueued` — items enqueued
- `DroppedNewest` — items dropped from the newest end
- `DroppedOldest` — items dropped from the oldest end
- `RetryEvents` — retry events triggered
- `ItemsWithRetry` — distinct items retried
- `MaxItemRetryAttempts` — highest retry count for any item

When a node uses observability, parallel strategies also publish retry annotations and activity tags.

## Thread Safety During Parallel Execution

When using parallel strategies:

- **Default profile**: context dictionaries are thread-safe (`ConcurrentDictionary`).
- **HighThroughput profile**: context dictionaries are plain `Dictionary` and are NOT thread-safe. Use `IPipelineStateManager` for shared state.

```csharp
public interface IPipelineStateManager
{
    ValueTask CreateSnapshotAsync(PipelineContext context, CancellationToken ct, bool forceFullSnapshot = false);
    ValueTask<bool> TryRestoreAsync(PipelineContext context, CancellationToken ct);
    void MarkNodeCompleted(string nodeId, PipelineContext context);
    void MarkNodeError(string nodeId, PipelineContext context);
}
```

Node restart and node retry don't roll back shared state; call `TryRestoreAsync` from your own recovery logic when a rollback is needed.

## Ordering

`PreserveOrdering = true` (the default) keeps input order in the output via a reorder buffer. Set `AllowUnorderedOutput()` to skip the reorder buffer, which removes head-of-line blocking and increases throughput when order is not important.

## Combining with Resilience

Parallel strategies use the same per-item executor as the sequential strategy, so item retry, backoff, the circuit breaker, and `OnItemFailure` behave the same way. Parallel strategies implement `IResumableExecutionStrategy`, so node restart works with them. You don't wrap the strategy yourself; the builder wraps it when restart is enabled for the node:

```csharp
var transform = builder.AddTransform<MyTransform, In, Out>("transform");
transform.RunParallel(builder, ParallelWorkloadType.IoBound);

builder.WithResilience(transform, options => options with
{
    ItemRetry = ItemRetryOptions.Default with { MaxRetries = 5 },
    NodeRestart = new NodeRestartOptions { MaxRestarts = 3 },
});
```

A restart resumes from the node's checkpoint. With ordered output, each output is delivered exactly once across restarts. With unordered output or a dropping queue policy, delivery is at least once.
