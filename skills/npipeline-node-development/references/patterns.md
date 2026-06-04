# Node Development Patterns

## ValueTask Fast Path

The per-item execution loop can avoid `Task` allocations when transforms are synchronous. Override `ExecuteValueTaskAsync` instead of `TransformAsync`:

```csharp
// DO THIS (allocation-free for synchronous work):
protected override ValueTask<EnrichedOrder> ExecuteValueTaskAsync(
    Order item, PipelineContext context, CancellationToken ct)
{
    var result = Enrich(item); // synchronous work
    return new ValueTask<EnrichedOrder>(result);
}

// NOT THIS (creates Task allocation per item):
public override Task<EnrichedOrder> TransformAsync(Order item, ...)
{
    var result = Enrich(item);
    return Task.FromResult(result); // heap allocation per item
}
```

When `ExecuteValueTaskAsync` is overridden, execution strategies that support `IValueTaskTransform` use it directly, bypassing `TransformAsync` entirely.

## Resource Disposal

All nodes implement `IAsyncDisposable`. Override when holding resources:

```csharp
public class DbSink : SinkNode<EnrichedOrder>
{
    private readonly SqlConnection _conn;

    public DbSink(SqlConnection conn) => _conn = conn;

    public override async ValueTask DisposeAsync()
    {
        await _conn.DisposeAsync();      // Dispose your resources
        await base.DisposeAsync();       // Always call base (calls GC.SuppressFinalize)
    }
}
```

For synchronous `IDisposable` resources (e.g., `FileStream`), use `DisposeAsync()` and convert:

```csharp
public override async ValueTask DisposeAsync()
{
        _fileStream?.Dispose();
    await base.DisposeAsync();
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
}
```

The factory uses compiled expression trees for constructor invocation, with `ActivatorUtilities` as a fallback. Greedy parameter resolution is used — the factory resolves each parameter from the `IServiceProvider` in order.

## Forwarding CancellationToken

Always pass `CancellationToken` to async operations:

```csharp
public override async Task<EnrichedOrder> TransformAsync(
    Order item, PipelineContext context, CancellationToken ct)
{
    var result = await _api.GetAsync(item.Id, ct);  // Pass ct
    return result;
}
```

For `IAsyncEnumerable` enumeration, use `.WithCancellation(ct)`:

```csharp
public override IDataStream<Order> OpenStream(
    PipelineContext context, CancellationToken ct)
{
    return DataStream.FromAsyncEnumerable(async cancellationToken =>
    {
        await foreach (var item in _reader.ReadAsync().WithCancellation(cancellationToken))
            yield return item;
    });
}
```

## INodeTypeMetadata

`TransformNode<TIn, TOut>` and `IStreamTransformNode<TIn, TOut>` implement `INodeTypeMetadata`, which provides `InputType` and `OutputType` as `Type` properties without reflection. The base class handles this automatically.

## Metadata Attributes

The `NPipeline.Attributes` namespace provides node metadata attributes:

| Attribute | Purpose |
|---|---|
| `[KeySelector(Type)]` | Marks the key selector property for join/aggregate nodes |
| `[MergeStrategy(Type)]` | Specifies merge strategy for fan-in nodes |
| `[Cardinality(int)]` | Declares expected cardinality |
| `[NodeOwner(string)]` | Tags node ownership |
| `[NodeDescription(string)]` | Human-readable node description |
| `[PipelineName(string)]` | Pipeline-level name override |

## Execution Strategy

Every transform and source node has an `ExecutionStrategy` property. Default: `SequentialExecutionStrategy`. Set directly or via the builder:

```csharp
// In node constructor
public MyNode() => ExecutionStrategy = new BatchingExecutionStrategy(100);

// Via builder
handle.WithExecutionStrategy(builder, new BatchingExecutionStrategy(100));
```
