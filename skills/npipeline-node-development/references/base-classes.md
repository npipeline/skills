# Base Classes Reference

## TransformNode<TIn, TOut>

The most common custom node. Extend this and implement `TransformAsync`.

```csharp
using NPipeline.Nodes;
using NPipeline.Pipeline;

public abstract class TransformNode<TIn, TOut>
    : ITransformNode<TIn, TOut>, INodeTypeMetadata, IValueTaskTransform<TIn, TOut>
{
    public Type InputType => typeof(TIn);        // No-reflection metadata
    public Type OutputType => typeof(TOut);

    // Default: SequentialExecutionStrategy — override or set via builder
    public IExecutionStrategy ExecutionStrategy { get; set; }

    // Required: your transformation logic
    public abstract Task<TOut> TransformAsync(
        TIn item, PipelineContext context, CancellationToken cancellationToken);

    // Optional: override for allocation-free synchronous transforms
    // Default wraps TransformAsync, so Task-based implementations already work.
    // Override to return a naturally produced ValueTask<TOut> to avoid allocations.
    ValueTask<TOut> IValueTaskTransform<TIn, TOut>.ExecuteValueTaskAsync(
        TIn item, PipelineContext context, CancellationToken cancellationToken)
    {
        return ExecuteValueTaskAsync(item, context, cancellationToken);
    }

    // Override in derived classes
    protected virtual ValueTask<TOut> ExecuteValueTaskAsync(
        TIn item, PipelineContext context, CancellationToken cancellationToken)
    {
        // Default: wrap the Task result
        return new ValueTask<TOut>(TransformAsync(item, context, cancellationToken));
    }

    // Disposal
    public virtual ValueTask DisposeAsync()
    {
        GC.SuppressFinalize(this);
        return ValueTask.CompletedTask;
    }
}
```

### Example: Synchronous Transform (ValueTask fast path)

```csharp
public class ValidateOrder : TransformNode<Order, Order>
{
    protected override ValueTask<Order> ExecuteValueTaskAsync(
        Order item, PipelineContext context, CancellationToken ct)
    {
        if (string.IsNullOrEmpty(item.CustomerName))
            throw new ValidationException("Customer name is required");
        item.ValidatedAt = DateTime.UtcNow;
        return new ValueTask<Order>(item);
    }

    // TransformAsync is NOT called when ExecuteValueTaskAsync is overridden
    public override Task<Order> TransformAsync(Order item, PipelineContext ctx, CancellationToken ct)
        => throw new NotImplementedException();
}
```

### Example: Asynchronous Transform

```csharp
public class EnrichOrder : TransformNode<Order, EnrichedOrder>
{
    private readonly ICustomerApi _api;

    public EnrichOrder(ICustomerApi api) => _api = api;

    public override async Task<EnrichedOrder> TransformAsync(
        Order item, PipelineContext context, CancellationToken ct)
    {
        var customer = await _api.GetCustomerAsync(item.CustomerId, ct);
        return new EnrichedOrder(item, customer);
    }
}
```

## SourceNode<TOut>

```csharp
public abstract class SourceNode<TOut> : ISourceNode<TOut>
{
    public IExecutionStrategy ExecutionStrategy { get; set; }
    public abstract IDataStream<TOut> OpenStream(
        PipelineContext context, CancellationToken cancellationToken);
}
```

### Example

```csharp
public class ApiSource : SourceNode<Order>
{
    private readonly IOrderRepository _repo;

    public ApiSource(IOrderRepository repo) => _repo = repo;

    public override IDataStream<Order> OpenStream(
        PipelineContext context, CancellationToken ct)
    {
        return DataStream.FromAsyncEnumerable(async cancellationToken =>
        {
            var cursor = 0;
            while (true)
            {
                var batch = await _repo.GetBatchAsync(cursor, pageSize: 100, cancellationToken);
                if (batch.Length == 0) yield break;
                foreach (var order in batch) yield return order;
                cursor += batch.Length;
            }
        });
    }
}
```

## SinkNode<TIn>

```csharp
public abstract class SinkNode<TIn> : ISinkNode<TIn>
{
    public abstract Task ConsumeAsync(
        IDataStream<TIn> input, PipelineContext context, CancellationToken cancellationToken);
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

    public override async ValueTask DisposeAsync()
    {
        await _connection.DisposeAsync();
        await base.DisposeAsync();
    }
}
```

## IStreamTransformNode<TIn, TOut>

Use when you need to process the entire stream, not individual items.

```csharp
public interface IStreamTransformNode<TIn, TOut> : IStreamTransformNode, INodeTypeMetadata
{
    IExecutionStrategy ExecutionStrategy { get; set; }
    IAsyncEnumerable<TOut> TransformAsync(
        IAsyncEnumerable<TIn> input,
        PipelineContext context,
        CancellationToken cancellationToken);
}
```

### Example: Sorting Node

```csharp
public class SortOrders : IStreamTransformNode<Order, Order>
{
    public Type InputType => typeof(Order);
    public Type OutputType => typeof(Order);
    public IExecutionStrategy ExecutionStrategy { get; set; } = new SequentialExecutionStrategy();

    public async IAsyncEnumerable<Order> TransformAsync(
        IAsyncEnumerable<Order> input,
        PipelineContext context,
        [EnumeratorCancellation] CancellationToken ct)
    {
        var items = await input.ToListAsync(ct);
        foreach (var item in items.OrderBy(o => o.CreatedAt))
            yield return item;
    }

    public ValueTask DisposeAsync() => ValueTask.CompletedTask;
}
```

Register via `builder.AddStreamTransform<SortOrders, Order, Order>()`.