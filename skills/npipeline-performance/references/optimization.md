# Performance Optimization Reference

## Optimization Profiles

### Default Profile

- `ConcurrentDictionary` for context dictionaries (thread-safe by default)
- Auto-configured: 3 retries, exponential backoff + full jitter, 10K materialization cap
- Performance analyzers NP9103-NP9107 suppressed
- Good for: prototyping, development, low-to-medium throughput

### HighThroughput Profile

- Pooled `Dictionary` instances (zero locking, zero allocation for dict ops)
- No automatic retries — user must explicitly configure everything
- All performance analyzers active
- Thread safety is the user's responsibility with plain `Dictionary`
- Good for: production pipelines processing millions of items/second

```csharp
builder.WithOptimizationProfile(PipelineOptimizationProfile.HighThroughput);
```

## Execution Strategies

```csharp
public interface IExecutionStrategy
{
    Task<IDataStream<TOut>> ExecuteAsync<TIn, TOut>(
        IDataStream<TIn> input,
        ITransformNode<TIn, TOut> node,
        PipelineContext context,
        CancellationToken cancellationToken);
}
```

### SequentialExecutionStrategy (default)

Processes one item at a time, in order. Uses `ValueTask` fast path when the node implements `IValueTaskTransform`.

### BatchingExecutionStrategy

Buffers items into `IReadOnlyList<TIn>` batches before passing to the transform:

```csharp
handle.WithExecutionStrategy(builder, new BatchingExecutionStrategy(batchSize: 100));
```

The transform receives `IReadOnlyList<T>` instead of individual items. Useful for bulk database inserts, API batch calls, or any operation with per-batch overhead.

### UnbatchingExecutionStrategy

Reverses batching — flattens `IReadOnlyList<T>` back to individual `T` items:

```csharp
handle.WithExecutionStrategy(builder, new UnbatchingExecutionStrategy());
```

### ResilientExecutionStrategy

Wraps another strategy with retry/restart/circuit-breaking logic. Automatically detects and materializes forward-only streams for replay:

```csharp
handle.WithResilience(builder);  // shorthand to wrap with resilient strategy
```

## Execution Plan Caching

`PipelineGraph` computes a SHA256 hash of its structure. The `IPipelineExecutionPlanCache` stores pre-built execution plans keyed by graph hash, avoiding rebuild costs on repeated runs. Default implementation: `InMemoryPipelineExecutionPlanCache`.

## Object Pooling

The `HighThroughput` profile uses pooled dictionaries from an internal object pool to avoid allocations on every pipeline run.

## Anti-Patterns

| Don't | Why | Instead |
|---|---|---|
| `source.ToListAsync()` in a transform | Materializes entire stream in memory | Process items one at a time |
| LINQ in `TransformAsync` | Allocates closures and iterators per item | Use direct loops |
| `Task.FromResult()` per item | Heap allocation per item | Override `ExecuteValueTaskAsync` |
| String concatenation in loops | Allocates intermediate strings | Use `StringBuilder` or pre-allocate |
| Anonymous objects per item | Heap allocation per item | Use value types or pooled objects |
| Blocking `.Result` or `.Wait()` | Deadlocks the async pipeline | Use `await` everywhere |
