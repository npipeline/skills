# Performance Optimization Reference

## Optimization Profiles

### Default Profile

- `ConcurrentDictionary` for context dictionaries (thread-safe by default)
- Auto-configured resilience: `ItemRetryOptions.Default` (3 item retries, exponential backoff 200 ms to 30 s with full jitter)
- Performance analyzers NP9103-NP9107 suppressed
- Good for: prototyping, development, low-to-medium throughput

### HighThroughput Profile

- Pooled `Dictionary` instances (zero locking, zero allocation for dictionary ops)
- No automatic retries: `PipelineResilienceOptions.None`
- All performance analyzers active
- Thread safety is the user's responsibility with plain `Dictionary`
- Good for: production pipelines processing millions of items/second

```csharp
builder.WithOptimizationProfile(PipelineOptimizationProfile.HighThroughput);
```

## ValueTask-Native Transforms

`TransformAsync` returns `ValueTask<TOut>`. A transform that completes synchronously allocates nothing per item. There is no `IValueTaskTransform` interface or `ExecuteValueTaskAsync` override:

```csharp
public override ValueTask<Result> TransformAsync(
    Input item, PipelineContext context, CancellationToken ct)
{
    return ValueTask.FromResult(new Result(item.Value * 2));   // no allocation
}
```

> [!TIP]
> An `async` method with no real `await` still allocates a state machine. Prefer `ValueTask.FromResult` for purely CPU-bound work.

## Execution Strategies

```csharp
public interface IExecutionStrategy
{
    Task<IDataStream<TOut>> ExecuteAsync<TIn, TOut>(
        IDataStream<TIn> input,
        ITransformNode<TIn, TOut> node,
        PipelineContext context,
        string nodeId,
        CancellationToken cancellationToken);
}
```

Note the explicit `string nodeId` parameter: the strategy is passed the node it is running, so it never has to ask the shared context.

Set a strategy on the graph (not the node):

```csharp
handle.WithExecutionStrategy(builder, new BatchingExecutionStrategy(batchSize: 100));
```

### SequentialExecutionStrategy (default)

Processes one item at a time, in order.

### BatchingExecutionStrategy

Buffers items into `IReadOnlyList<TIn>` batches before passing to the transform, which then receives `IReadOnlyList<T>`. Useful for bulk database inserts or API batch calls.

### UnbatchingExecutionStrategy

Reverses batching: flattens `IReadOnlyList<T>` back to individual `T` items.

### ResilientExecutionStrategy

`internal` and applied automatically. When `NodeRestart.MaxRestarts` is above zero for a transform, the builder wraps its strategy for restart — there is nothing to call. See the `npipeline-resilience` skill.

## A Node's Default Strategy

A node type with an inherent default strategy implements `IExecutionStrategyProvider`:

```csharp
public IExecutionStrategy DefaultExecutionStrategy => new BatchingExecutionStrategy(100);
```

A strategy configured on the graph with `WithExecutionStrategy` takes precedence.

## Execution Plan Caching

`PipelineGraph` supports execution plan caching. The `IPipelineExecutionPlanCache` stores pre-built execution plans keyed by graph topology, avoiding rebuild costs on repeated runs. Default implementation: `InMemoryPipelineExecutionPlanCache`.

## Object Pooling

The `HighThroughput` profile uses pooled dictionaries to avoid allocations on every pipeline run.

## Anti-Patterns

| Don't | Why | Instead |
|---|---|---|
| `source.ToListAsync()` in a transform | Materializes entire stream in memory | Process items one at a time |
| LINQ in `TransformAsync` | Allocates closures and iterators per item | Use direct loops |
| `Task.FromResult()` per item | Heap allocation per item | Return `ValueTask.FromResult(...)` |
| String concatenation in loops | Allocates intermediate strings | Use `StringBuilder` or pre-allocate |
| Anonymous objects per item | Heap allocation per item | Use value types or pooled objects |
| Blocking `.Result` or `.Wait()` | Deadlocks the async pipeline | Use `await` everywhere |
