# Base Classes Reference

## TransformNode<TIn, TOut>

The most common custom node. Extend this and implement `TransformAsync`.

```csharp
using NPipeline.Nodes;
using NPipeline.Pipeline;

public abstract class TransformNode<TIn, TOut>
    : ITransformNode<TIn, TOut>, INodeTypeMetadata
{
    public Type InputType => typeof(TIn);        // No-reflection metadata
    public Type OutputType => typeof(TOut);

    // Returns ValueTask, so a synchronous transform allocates nothing per item.
    public abstract ValueTask<TOut> TransformAsync(
        TIn item, PipelineContext context, CancellationToken cancellationToken);
}
```

How the node runs is a property of the graph, not the node: the execution strategy lives on the graph node definition and is set with `WithExecutionStrategy`. `TransformNode<TIn, TOut>` has no `ExecutionStrategy` property.

### Example: Synchronous Transform

Return `ValueTask.FromResult(...)` directly. There is no separate fast path to override.

```csharp
public class ValidateOrder : TransformNode<Order, Order>
{
    public override ValueTask<Order> TransformAsync(
        Order item, PipelineContext context, CancellationToken ct)
    {
        if (string.IsNullOrEmpty(item.CustomerName))
            throw new ValidationException("Customer name is required");

        item.ValidatedAt = DateTime.UtcNow;
        return ValueTask.FromResult(item);
    }
}
```

### Example: Asynchronous Transform

An `async` method returning `ValueTask<TOut>` needs no other change.

```csharp
public class EnrichOrder : TransformNode<Order, EnrichedOrder>
{
    private readonly ICustomerApi _api;

    public EnrichOrder(ICustomerApi api) => _api = api;

    public override async ValueTask<EnrichedOrder> TransformAsync(
        Order item, PipelineContext context, CancellationToken ct)
    {
        var customer = await _api.GetCustomerAsync(item.CustomerId, ct);
        return new EnrichedOrder(item, customer);
    }
}
```

## SourceNode<TOut>

```csharp
public abstract class SourceNode<TOut> : ISourceNode<TOut>, INodeTypeMetadata
{
    public Type? InputType => null;
    public Type OutputType => typeof(TOut);

    public abstract IDataStream<TOut> OpenStream(
        PipelineContext context, CancellationToken cancellationToken);

    // Attributes dead-lettered rows to this node; call from OpenStream.
    protected DeadLetterChannel OpenDeadLetterChannel(PipelineContext context);
}
```

### Example

Return a `DataStream<T>` wrapping an `IAsyncEnumerable<T>` for lazy streaming. Use `InMemoryDataStream<T>` only for small, bounded collections.

```csharp
using NPipeline.DataFlow.DataStreams;

public class ApiSource : SourceNode<Order>
{
    private readonly IOrderRepository _repo;

    public ApiSource(IOrderRepository repo) => _repo = repo;

    public override IDataStream<Order> OpenStream(
        PipelineContext context, CancellationToken ct)
    {
        return new DataStream<Order>(ReadAsync(ct), "orders");
    }

    private async IAsyncEnumerable<Order> ReadAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        var cursor = 0;

        while (true)
        {
            var batch = await _repo.GetBatchAsync(cursor, pageSize: 100, cancellationToken);

            if (batch.Length == 0)
                yield break;

            foreach (var order in batch)
                yield return order;

            cursor += batch.Length;
        }
    }
}
```

## SinkNode<TIn>

```csharp
public abstract class SinkNode<TIn> : ISinkNode<TIn>, INodeTypeMetadata
{
    public Type InputType => typeof(TIn);
    public Type? OutputType => null;

    public abstract Task ConsumeAsync(
        IDataStream<TIn> input, PipelineContext context, CancellationToken cancellationToken);

    // Attributes dead-lettered writes to this node; call at the start of ConsumeAsync.
    protected DeadLetterChannel OpenDeadLetterChannel(PipelineContext context);
}
```

### Example

```csharp
public class DatabaseSink : SinkNode<EnrichedOrder>, IAsyncDisposable
{
    private readonly DbConnection _connection;

    public DatabaseSink(DbConnection connection) => _connection = connection;

    public override async Task ConsumeAsync(
        IDataStream<EnrichedOrder> input, PipelineContext context, CancellationToken ct)
    {
        await foreach (var item in input.WithCancellation(ct))
        {
            await _connection.ExecuteAsync(
                "INSERT INTO orders ...", item, cancellationToken: ct);
        }
    }

    // Nodes are disposable only if they implement it; there is no base to call.
    public async ValueTask DisposeAsync() => await _connection.DisposeAsync();
}
```

> [!WARNING]
> You must consume the `input` parameter in `ConsumeAsync`. The `SinkNodeInputConsumptionAnalyzer` (NP9301) reports an error if you don't.

## IStreamTransformNode<TIn, TOut>

Use when you need to process the entire stream, not individual items.

```csharp
public interface IStreamTransformNode<in TIn, TOut> : IStreamTransformNode
{
    IAsyncEnumerable<TOut> TransformAsync(
        IAsyncEnumerable<TIn> items,
        PipelineContext context,
        CancellationToken cancellationToken);
}
```

The interface has no `ExecutionStrategy` property and does not implement `INodeTypeMetadata`. Register via `builder.AddStreamTransform<SortOrders, Order, Order>()`.

### Example: Sorting Node

```csharp
public class SortOrders : IStreamTransformNode<Order, Order>
{
    public async IAsyncEnumerable<Order> TransformAsync(
        IAsyncEnumerable<Order> items,
        PipelineContext context,
        [EnumeratorCancellation] CancellationToken ct)
    {
        var buffer = new List<Order>();

        await foreach (var item in items.WithCancellation(ct))
            buffer.Add(item);

        buffer.Sort(static (a, b) => a.CreatedAt.CompareTo(b.CreatedAt));

        foreach (var item in buffer)
            yield return item;
    }
}
```

## LookupNode<TIn, TKey, TValue, TOut>

`LookupAsync` returns `ValueTask<TValue?>`, so a cache hit costs no allocation.

```csharp
public class CustomerLookup : LookupNode<Order, int, Customer, EnrichedOrder>
{
    protected override int ExtractKey(Order input, PipelineContext context)
        => input.CustomerId;

    protected override async ValueTask<Customer?> LookupAsync(
        int key, PipelineContext context, CancellationToken ct)
        => await _db.FindCustomerAsync(key, ct);

    protected override EnrichedOrder CreateOutput(
        Order input, Customer? customer, PipelineContext context)
        => new(input, customer);
}
```
