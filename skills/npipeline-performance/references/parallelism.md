# Parallel Execution Reference

## Package

```
dotnet add package NPipeline.Extensions.Parallelism
```

## Parallel Execution Strategies

Three strategies built on TPL Dataflow with configurable backpressure:

| Strategy | Backpressure | Use Case |
|---|---|---|
| `BlockingParallelStrategy` | Blocks producer when queue full | General-purpose, preserve ordering |
| `DropOldestParallelStrategy` | Drops oldest items when queue full | Real-time streaming (new > old) |
| `DropNewestParallelStrategy` | Drops newest items when queue full | Back-pressure sensitivity (old > new) |

## ParallelOptions

```csharp
public sealed record ParallelOptions(
    int MaxDegreeOfParallelism,
    int MaxQueueLength,
    BoundedQueuePolicy QueuePolicy,
    int OutputBufferCapacity,
    bool PreserveOrdering,
    TimeSpan? MetricsInterval)
```

### Workload Presets

```csharp
// Easiest API — choose a preset
handle.RunParallel(builder, ParallelWorkloadType.CpuBound);
handle.RunParallel(builder, ParallelWorkloadType.IoBound);
```

| Preset | MaxDoP | Max Queue | Queue Policy | Best For |
|---|---|---|---|---|
| `General` | Env.Processors | 1024 | Block | Default catch-all |
| `CpuBound` | Env.Processors | Env.Processors × 2 | Block | CPU-heavy transforms |
| `IoBound` | Env.Processors × 4 | 4096 | DropNewest | Network/DB calls |
| `NetworkBound` | Env.Processors × 8 | 8192 | DropOldest | High-latency external calls |

### Custom Configuration

```csharp
handle.WithBlockingParallelism(builder,
    maxDegreeOfParallelism: 8,
    maxQueueLength: 2048,
    outputBufferCapacity: 256);

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

handle.WithParallelism(builder, options, new BlockingParallelStrategy());
```

## Parallel Execution Metrics

When metrics are enabled (via `MetricsInterval`), the strategy tracks:

- `Processed` — Total items processed
- `Enqueued` — Total items enqueued
- `DroppedNewest` — Items dropped from newest end (DropNewest strategy)
- `DroppedOldest` — Items dropped from oldest end (DropOldest strategy)
- `RetryEvents` — Retry events triggered
- `ItemsWithRetry` — Distinct items that were retried
- `MaxItemRetryAttempts` — Highest retry count for any item

```csharp
var metrics = strategy.GetMetrics();
Console.WriteLine($"Processed: {metrics.Processed}, Dropped: {metrics.DroppedNewest}");
```

## Thread Safety During Parallel Execution

When using parallel strategies:

- **Default profile**: Context dictionaries are `ConcurrentDictionary` — safe for concurrent access.
- **HighThroughput profile**: Context dictionaries are plain `Dictionary` — NOT thread-safe. Use `IPipelineStateManager` for shared state:

```csharp
// In your node, use IPipelineStateManager instead of context.Items directly
public class MyNode : TransformNode<Order, Order>
{
    private readonly IPipelineStateManager _state;

    public override async Task<Order> TransformAsync(
        Order item, PipelineContext context, CancellationToken ct)
    {
        await _state.UpdateAsync("counter", (int c) => c + 1, ct);
        return item;
    }
}
```

## Ordering

Set `PreserveOrdering = true` to maintain input order in the output stream. Ordering adds overhead from TPL Dataflow's reorder buffer, so disable it when order is not important.
