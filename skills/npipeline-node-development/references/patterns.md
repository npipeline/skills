# Node Development Patterns

## ValueTask-Native Transforms

`TransformAsync` returns `ValueTask<TOut>`. A transform that completes synchronously allocates nothing per item. There is no separate interface to implement or method to override:

```csharp
// Allocation-free for synchronous work:
public override ValueTask<EnrichedOrder> TransformAsync(
    Order item, PipelineContext context, CancellationToken ct)
{
    var result = Enrich(item); // synchronous work
    return ValueTask.FromResult(result);
}

// Still valid, but allocates a state machine per item when it does not complete synchronously:
public override async ValueTask<EnrichedOrder> TransformAsync(
    Order item, PipelineContext context, CancellationToken ct)
{
    var result = await EnrichAsync(item, ct);
    return result;
}
```

> [!TIP]
> Prefer the synchronous `ValueTask.FromResult` form for CPU-only work. An `async` method with no real `await` allocates a state machine even when it returns a completed task.

## Resource Disposal

Nodes are disposable only if they implement `IAsyncDisposable` or `IDisposable` themselves. The runtime checks for the interface and disposes the instance at the end of the run that created it. Instances resolved from a DI container are left to the container, so they are not disposed twice. There is no base implementation to call:

```csharp
public class DbSink : SinkNode<EnrichedOrder>, IAsyncDisposable
{
    private readonly SqlConnection _conn;

    public DbSink(SqlConnection conn) => _conn = conn;

    public override async Task ConsumeAsync(
        IDataStream<EnrichedOrder> input, PipelineContext context, CancellationToken ct)
    {
        // ...
    }

    public async ValueTask DisposeAsync() => await _conn.DisposeAsync();
}
```

For a synchronous `IDisposable` resource, dispose it from `DisposeAsync`:

```csharp
public ValueTask DisposeAsync()
{
    _fileStream?.Dispose();
    return ValueTask.CompletedTask;
}
```

## Constructor Injection

Nodes instantiated through the `DiContainerNodeFactory` support constructor injection:

```csharp
public class MyTransform : TransformNode<Order, EnrichedOrder>
{
    private readonly IOrderService _svc;
    private readonly ILogger<MyTransform> _log;
    private readonly IOptions<MySettings> _opts;

    public MyTransform(IOrderService svc, ILogger<MyTransform> log, IOptions<MySettings> opts)
    {
        _svc = svc;
        _log = log;
        _opts = opts;
    }

    public override async ValueTask<EnrichedOrder> TransformAsync(
        Order item, PipelineContext context, CancellationToken ct)
    {
        var enriched = await _svc.EnrichAsync(item, ct);
        _log.LogInformation("Enriched order {Id}", item.Id);
        return enriched;
    }
}
```

The factory uses compiled expression trees for constructor invocation, with `ActivatorUtilities` as a fallback. Greedy parameter resolution is used: the factory resolves each parameter from the `IServiceProvider` in order.

## Forwarding CancellationToken

Always pass `CancellationToken` to async operations:

```csharp
public override async ValueTask<EnrichedOrder> TransformAsync(
    Order item, PipelineContext context, CancellationToken ct)
{
    var result = await _api.GetAsync(item.Id, ct);  // Pass ct
    return result;
}
```

For `IAsyncEnumerable` enumeration, use `.WithCancellation(ct)`:

```csharp
await foreach (var item in input.WithCancellation(ct))
{
    // ...
}
```

## INodeTypeMetadata

`TransformNode<TIn, TOut>`, `SourceNode<TOut>`, `SinkNode<TIn>`, and `LookupNode<...>` implement `INodeTypeMetadata`, which provides `InputType` and `OutputType` as `Type` properties without reflection. The base class handles this automatically. `IStreamTransformNode<TIn, TOut>` does **not** implement `INodeTypeMetadata`; the framework derives its input and output types from the interface's generic arguments.

## Metadata Attributes

The `NPipeline.Attributes` namespace provides node metadata attributes:

| Attribute | Purpose |
|---|---|
| `[KeySelector(Type, params string[])]` | Marks the key property (or properties) for a given input type on join nodes. Apply once per input type. |
| `[MergeStrategy(Type)]` | Specifies merge strategy for fan-in nodes |
| `[TransformCardinality(TransformCardinality)]` | Declares a transform's input-to-output cardinality (for example `OneToOne`, `OneToMany`) |
| `[NodeOwner(string)]` | Tags node ownership |
| `[NodeDescription(string)]` | Human-readable node description |
| `[NodeRemark(string)]` | Additional documentation remark |
| `[PipelineName(string)]` / `[PipelineDescription(string)]` | Pipeline-level metadata |
| `[LineageMapper(Type)]` | Declares the lineage mapper used when the node reshapes items |

## Execution Strategy

There is no `ExecutionStrategy` property on a node. Set the strategy on the graph through the handle:

```csharp
// Via the fluent handle extension
handle.WithExecutionStrategy(builder, new BatchingExecutionStrategy(100));
```

A node type with an inherent default strategy implements `IExecutionStrategyProvider`:

```csharp
public class MyBatchingNode : IStreamTransformNode<Order, IReadOnlyCollection<Order>>, IExecutionStrategyProvider
{
    public IExecutionStrategy DefaultExecutionStrategy => new BatchingExecutionStrategy(100);

    // ...
}
```

A strategy configured on the graph with `WithExecutionStrategy` takes precedence over the node's supplied default.
