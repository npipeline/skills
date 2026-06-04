# PipelineBuilder API Reference

The `PipelineBuilder` is the central fluent API for constructing pipeline graphs. It enforces compile-time type safety through typed node handles.

## Node Registration Methods

All methods accept an optional `string? name` parameter. If omitted, the node type name is used (with deduplication).

### Sources

```csharp
// Compile-time typed (preferred)
SourceNodeHandle<TOut> AddSource<TNode, TOut>(string? name = null)
    where TNode : ISourceNode<TOut>

// Runtime type (for scenarios where type is only known at runtime)
SourceNodeHandle<TOut> AddSource<TOut>(Type nodeType, string? name = null)
```

### Transforms

```csharp
// Item-at-a-time transform
TransformNodeHandle<TIn, TOut> AddTransform<TNode, TIn, TOut>(string? name = null)
    where TNode : ITransformNode<TIn, TOut>

// Whole-stream transform
TransformNodeHandle<TIn, TOut> AddStreamTransform<TNode, TIn, TOut>(string? name = null)
    where TNode : IStreamTransformNode<TIn, TOut>
```

### Sinks

```csharp
// Compile-time typed
SinkNodeHandle<TIn> AddSink<TNode, TIn>(string? name = null)
    where TNode : ISinkNode<TIn>

// Runtime type
SinkNodeHandle<TIn> AddSink<TIn>(Type nodeType, string? name = null)
```

### Joins

```csharp
JoinNodeHandle<TIn1, TIn2, TOut> AddJoin<TNode, TIn1, TIn2, TOut>(string? name = null)
    where TNode : IJoinNode
```

### Aggregation

```csharp
// Accumulator and result are different types
AggregateNodeHandle<TIn, TResult> AddAggregate<TNode, TIn, TKey, TAccumulate, TResult>(string? name = null)
    where TNode : IAggregateNode where TKey : notnull

// Accumulator and result are same type (simplified)
AggregateNodeHandle<TIn, TResult> AddAggregate<TNode, TIn, TKey, TResult>(string? name = null)
    where TNode : IAggregateNode where TKey : notnull
```

### Lambda Convenience Nodes

Provided by `PipelineBuilderExtensions`:

```csharp
// Source from collection
builder.AddSource(() => new[] { 1, 2, 3 });
// Async source
builder.AddSource(async ct => FetchDataAsync(ct));

// Sync transform (uses ValueTask fast path)
builder.AddTransform((string s) => s.ToUpper());
// Async transform
builder.AddTransform(async (item, ct) => await ProcessAsync(item, ct));

// Sync sink
builder.AddSink((string s) => Console.WriteLine(s));
// Async sink
builder.AddSink(async (item, ct) => await SaveAsync(item, ct));
```

### Internal Override Methods

These are used by extension packages to register nodes with specific `NodeKind` overrides (Tap, Branch, Lookup, Batcher, Unbatcher, CompositeInput, CompositeOutput):

```csharp
internal TransformNodeHandle<TIn, TOut> AddTransformWithKind<TNode, TIn, TOut>(NodeKind kind, string? name)
internal SourceNodeHandle<TOut> AddSourceWithKind<TNode, TOut>(NodeKind kind, string? name)
internal SinkNodeHandle<TIn> AddSinkWithKind<TNode, TIn>(NodeKind kind, string? name)
internal TransformNodeHandle<TIn, TOut> AddStreamTransformWithKind<TNode, TIn, TOut>(NodeKind kind, string? name)
```

## Connecting Nodes

```csharp
// Simple typed connection — compiler enforces matching types
builder.Connect(sourceHandle, transformHandle);
builder.Connect(transformHandle, sinkHandle);

// Join connections — overloads match input positions
builder.Connect(source1, joinHandle); // connects to TIn1
builder.Connect(source2, joinHandle); // connects to TIn2
```

A single source handle can be connected to multiple targets (fan-out). Each target receives every item.

## Node Handle Types

Typed handles prevent connecting incompatible nodes at compile time:

| Handle Type | Parameters | Interfaces |
|---|---|---|
| `SourceNodeHandle<TOut>` | `TOut` | `IOutputNodeHandle<TOut>` |
| `TransformNodeHandle<TIn, TOut>` | `TIn`, `TOut` | `IInputNodeHandle<TIn>`, `IOutputNodeHandle<TOut>` |
| `SinkNodeHandle<TIn>` | `TIn` | `IInputNodeHandle<TIn>` |
| `JoinNodeHandle<TIn1, TIn2, TOut>` | `TIn1`, `TIn2`, `TOut` | `IInputNodeHandle<TIn1>` (port 1), `IInputNodeHandle<TIn2>` (port 2), `IOutputNodeHandle<TOut>` |
| `AggregateNodeHandle<TIn, TResult>` | `TIn`, `TResult` | `IInputNodeHandle<TIn>`, `IOutputNodeHandle<TResult>` |

## Configuration Methods

### Resilience and Error Handling

```csharp
// Retry options (global)
builder.WithRetryOptions(o => o with { MaxItemRetries = 3 });
// Per-node retry
builder.WithRetryOptions(handle, myRetryOptions);

// Circuit breaker
builder.WithCircuitBreaker(failureThreshold: 5, openDuration: TimeSpan.FromMinutes(1));

// Resilience policy
builder.AddResiliencePolicy<MyPolicy>();

// Dead letter sink
builder.AddDeadLetterSink<MyDeadLetterSink>();
```

### Execution Strategy (per-node)

```csharp
builder.WithExecutionStrategy(handle, new BatchingExecutionStrategy(100));
// or via handle extension
handle.WithExecutionStrategy(builder, new BatchingExecutionStrategy(100));
```

### Optimization Profile

```csharp
builder.WithOptimizationProfile(PipelineOptimizationProfile.HighThroughput);
```

### Lineage

```csharp
builder.EnableItemLevelLineage(opts => opts with { SampleEvery = 10 });
```

## Building

```csharp
// Validates graph, throws on error by default
Pipeline pipeline = builder.Build();

// Non-throwing variant
if (builder.TryBuild(out var pipeline, out var validationResult))
{
    // pipeline is valid
}
else
{
    // inspect validationResult.Errors
}
```

`Build()` validates the graph, computes a SHA256 graph hash for execution plan caching, and builds child graphs for any composite nodes.

## Complete Example

```csharp
public class OrderPipeline : IPipelineDefinition
{
    public void Define(PipelineBuilder builder, PipelineContext context)
    {
        var source   = builder.AddSource<CsvSourceNode<Order>, Order>("read-orders");
        var validate = builder.AddTransform<ValidateOrder, Order, Order>("validate");
        var enrich   = builder.AddTransform<EnrichOrder, Order, EnrichedOrder>("enrich");
        var save     = builder.AddSink<DatabaseSink, EnrichedOrder>("save");

        builder.Connect(source, validate);
        builder.Connect(validate, enrich);
        builder.Connect(enrich, save);
    }
}

var runner = PipelineRunner.Create();
await runner.RunAsync<OrderPipeline>();
```
