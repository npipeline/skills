---
name: npipeline-performance
description: Use when the user wants to optimize NPipeline pipeline performance. Covers optimization profiles (Default vs HighThroughput), ValueTask fast paths, execution plan caching, execution strategies (Sequential, Batching, Resilient), zero-allocation hot paths, and object pooling. Use when user mentions "optimize", "make it faster", "high throughput", "ValueTask", "execution strategy", "performance profile", or "zero allocation".
npipelineVersion: "0.52.0"
lastVerified: "2026-06-04"
---

# NPipeline Performance

This skill covers optimizing NPipeline pipeline performance: optimization profiles, execution strategies, parallel execution, and zero-allocation patterns.

## Workflow

When the user wants to improve performance, work through these layers from simplest to most involved:

### Layer 1: Choose the Right Optimization Profile

```csharp
builder.WithOptimizationProfile(PipelineOptimizationProfile.HighThroughput);
```

| Profile | Dictionaries | Retries | Memory | Use Case |
|---|---|---|---|---|
| `Default` | `ConcurrentDictionary` (thread-safe) | Auto 3 retries + jitter | Higher overhead | Prototyping, low-medium throughput |
| `HighThroughput` | Pooled `Dictionary` (zero-lock) | None (explicit only) | Minimal overhead | Millions of items/second |

`HighThroughput` also activates all Roslyn analyzer rules (NP9103-NP9107) for build-time performance checks.

### Layer 2: Use ValueTask Fast Paths

Override `ExecuteValueTaskAsync` for synchronous transforms to avoid per-item `Task` allocations:

```csharp
protected override ValueTask<EnrichedOrder> ExecuteValueTaskAsync(
    Order item, PipelineContext ctx, CancellationToken ct)
{
    var result = Enrich(item); // No async allocation
    return new ValueTask<EnrichedOrder>(result);
}
```

The execution loop checks for `IValueTaskTransform` and uses the fast path directly, eliminating allocations for every item in synchronous transforms.

### Layer 3: Apply Execution Strategies

Every node has an `IExecutionStrategy` that controls how it processes input:

| Strategy | When to Use |
|---|---|
| `SequentialExecutionStrategy` (default) | Simple, ordered processing |
| `BatchingExecutionStrategy(batchSize)` | Database bulk inserts, API batch calls |
| `UnbatchingExecutionStrategy` | Flatten batches back to items |
| `ResilientExecutionStrategy` | Adds retry/restart/circuit-breaking to another strategy |

```csharp
handle.WithExecutionStrategy(builder, new BatchingExecutionStrategy(100));
```

### Layer 4: Add Parallelism

For CPU-bound or I/O-bound workloads, use parallel execution strategies from `NPipeline.Extensions.Parallelism`. See `references/parallelism.md` for full details.

### Layer 5: Avoid Common Pitfalls

- Don't materialize in sources (let data stream lazily)
- Don't collect `IAsyncEnumerable` into lists in transforms
- Don't use LINQ in hot paths
- Don't create anonymous objects per item
- Forward `CancellationToken` to all async calls

Consult `references/optimization.md` for detailed performance patterns and anti-patterns.
