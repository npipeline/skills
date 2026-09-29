---
name: npipeline-performance
description: Use when the user wants to optimize NPipeline pipeline performance. Covers optimization profiles (Default vs HighThroughput), the ValueTask-native transform API, execution plan caching, execution strategies (Sequential, Batching, parallel), zero-allocation hot paths, and parallel execution with backpressure policies. Use when user mentions "optimize", "make it faster", "high throughput", "ValueTask", "execution strategy", "performance profile", or "zero allocation".
npipelineVersion: "0.67.0"
lastVerified: "2026-09-29"
---

# NPipeline Performance

This skill covers optimizing NPipeline pipeline performance: optimization profiles, execution strategies, parallel execution, and zero-allocation patterns.

## Workflow

When the user wants to improve performance, work through these layers from simplest to most involved.

### Layer 1: Choose the Right Optimization Profile

```csharp
builder.WithOptimizationProfile(PipelineOptimizationProfile.HighThroughput);
```

| Profile | Dictionaries | Retries | Memory | Use Case |
|---|---|---|---|---|
| `Default` | `ConcurrentDictionary` (thread-safe) | Item retry: 3 retries + jitter | Higher overhead | Prototyping, low-medium throughput |
| `HighThroughput` | Pooled `Dictionary` (zero-lock) | None (explicit only) | Minimal overhead | Millions of items/second |

`HighThroughput` also activates all Roslyn analyzer rules (NP9103-NP9107) for build-time performance checks.

### Layer 2: Use the ValueTask-Native Transform

`TransformAsync` returns `ValueTask<TOut>`, so a synchronous transform allocates nothing per item. Return `ValueTask.FromResult(...)` for CPU-only work; there is no separate fast-path interface to implement:

```csharp
public override ValueTask<EnrichedOrder> TransformAsync(
    Order item, PipelineContext ctx, CancellationToken ct)
{
    var result = Enrich(item); // synchronous work, no allocation
    return ValueTask.FromResult(result);
}
```

### Layer 3: Apply Execution Strategies

How a node runs is configured on the graph, not the node:

| Strategy | When to Use |
|---|---|
| `SequentialExecutionStrategy` (default) | Simple, ordered processing |
| `BatchingExecutionStrategy(batchSize)` | Database bulk inserts, API batch calls |
| `UnbatchingExecutionStrategy` | Flatten batches back to items |
| `ResilientExecutionStrategy` | Applied automatically from `NodeRestart` options; internal |

```csharp
handle.WithExecutionStrategy(builder, new BatchingExecutionStrategy(100));
```

### Layer 4: Add Parallelism

For CPU-bound or I/O-bound workloads, use parallel execution from `NPipeline.Extensions.Parallelism`. See `references/parallelism.md`.

### Layer 5: Avoid Common Pitfalls

- Don't materialize in sources (let data stream lazily)
- Don't collect `IAsyncEnumerable` into lists in transforms
- Don't use LINQ in hot paths
- Don't create anonymous objects per item
- Forward `CancellationToken` to all async calls

Consult `references/optimization.md` for detailed performance patterns and anti-patterns.
