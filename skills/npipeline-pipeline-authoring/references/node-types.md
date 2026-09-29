# Node Types Reference

## Node Kind Enum

The `NodeKind` enum classifies every node in the pipeline graph:

- **`Source`** — Produces data; has only output
- **`Transform`** — Consumes one item, produces one item
- **`StreamTransform`** — Consumes/produces whole streams (batching, unbatching, composition)
- **`Tap`** — Side-channel sink; copies items without affecting main flow
- **`Branch`** — Side-effect handler attached to a stream
- **`Route`** — Conditional fan-out to named destinations
- **`Lookup`** — Enriches items via key-based lookup
- **`Batch`** — Groups items into batches
- **`Sink`** — Terminal consumer; has only input
- **`Join`** — Combines two input streams into one output stream
- **`Aggregate`** — Computes grouped results over windows
- **`Composite`** — Wraps a sub-pipeline
- **`CompositeInput`** — Input bridge for composite pipelines
- **`CompositeOutput`** — Output bridge for composite pipelines

## Core Node Interfaces

### ISourceNode<TOut>

```csharp
public interface ISourceNode<out TOut> : INode
{
    IDataStream<TOut> OpenStream(PipelineContext context, CancellationToken cancellationToken);
}
```

Base class: `SourceNode<TOut>` — extend this, implement `OpenStream`.

### ITransformNode<TIn, TOut>

```csharp
public interface ITransformNode : INode { }

public interface ITransformNode<in TIn, TOut> : ITransformNode
{
    ValueTask<TOut> TransformAsync(TIn item, PipelineContext context, CancellationToken cancellationToken);
}
```

Base class: `TransformNode<TIn, TOut>` — extend this, implement `TransformAsync`. The base class exposes `InputType` and `OutputType` without reflection.

> [!IMPORTANT]
> `TransformAsync` returns `ValueTask<TOut>`, not `Task<TOut>`. There is no `IValueTaskTransform` and no `ExecuteValueTaskAsync` to override.

### IStreamTransformNode<TIn, TOut>

```csharp
public interface IStreamTransformNode : INode { }

public interface IStreamTransformNode<in TIn, TOut> : IStreamTransformNode
{
    IAsyncEnumerable<TOut> TransformAsync(
        IAsyncEnumerable<TIn> items,
        PipelineContext context,
        CancellationToken cancellationToken);
}
```

Use when you need to process the entire stream as a whole (e.g., sorting, batching). Unlike the other core interfaces, this one does not implement `INodeTypeMetadata`.

### ISinkNode<TIn>

```csharp
public interface ISinkNode<in TIn> : INode
{
    Task ConsumeAsync(IDataStream<TIn> input, PipelineContext context, CancellationToken cancellationToken);
}
```

Base class: `SinkNode<TIn>` — extend this, implement `ConsumeAsync`.

### IJoinNode

```csharp
public interface IJoinNode : INode { }
```

Joins combine two input streams of types `TIn1` and `TIn2` into an output stream of type `TOut`. Base class: `BaseJoinNode<TKey, TIn1, TIn2, TOut>`.

### IAggregateNode

```csharp
public interface IAggregateNode : INode { }
```

Base class: `AggregateNode<TIn, TKey, TResult>` or `AdvancedAggregateNode<TIn, TKey, TAccumulate, TResult>`.

### IExecutionStrategyProvider

A node type with an inherent default execution strategy implements this; a strategy configured on the graph with `WithExecutionStrategy` takes precedence.

```csharp
public interface IExecutionStrategyProvider
{
    IExecutionStrategy DefaultExecutionStrategy { get; }
}
```

## Common Built-in Node Classes

| Node Class | Kind | Purpose |
|---|---|---|
| `LambdaSourceNode<TOut>` | Source | Inline source from delegate |
| `LambdaTransformNode<TIn, TOut>` | Transform | Inline sync transform from delegate |
| `AsyncLambdaTransformNode<TIn, TOut>` | Transform | Inline async transform from delegate |
| `LambdaSinkNode<TIn>` | Sink | Inline sink from delegate |
| `FilterNode<T>` | StreamTransform | Passes items satisfying a predicate |
| `SelectManyNode<TIn, TOut>` | StreamTransform | Expands one item into zero or more |
| `KeyedJoinNode<TKey, TIn1, TIn2, TOut>` | Join | Key-based equi-join (TKey is first type param) |
| `TimeWindowedJoinNode<TKey, TIn1, TIn2, TOut>` | Join | Time-windowed join (TKey is first type param) |
| `AggregateNode<TIn, TKey, TResult>` | Aggregate | Grouped aggregation (TAccumulate = TResult) |
| `AdvancedAggregateNode<TIn, TKey, TAccumulate, TResult>` | Aggregate | Accumulator ≠ result type |
| `LookupNode<TIn, TKey, TValue, TOut>` | Lookup | Custom lookup transform |
| `RouteNode<T>` | Route | Conditional routing (use `AddRoute` + `ConnectWhen`) |
| `BranchNode<T>` | Branch | Side-effect branching |
| `TapNode<T>` | Tap | Side-channel monitoring |
| `BatchingNode<T>` | Batch | Groups items (outputs `IReadOnlyCollection<T>`) |
| `UnbatchingNode<T>` / `ReadOnlyCollectionUnbatchingNode<T>` | Batch | Flattens batches back to individual items |
| `CustomMergeNode<TIn>` | Transform | Custom stream merging |
