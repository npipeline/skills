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
public interface ISourceNode<TOut> : INode
{
    IExecutionStrategy ExecutionStrategy { get; set; }
    IDataStream<TOut> OpenStream(PipelineContext context, CancellationToken cancellationToken);
}
```

Base class: `SourceNode<TOut>` — extend this, implement `OpenStream`.

### ITransformNode<TIn, TOut>

```csharp
public interface ITransformNode<TIn, TOut> : INode, INodeTypeMetadata
{
    IExecutionStrategy ExecutionStrategy { get; set; }
    Task<TOut> TransformAsync(TIn item, PipelineContext context, CancellationToken cancellationToken);
}
```

Base class: `TransformNode<TIn, TOut>` — extend this, implement `TransformAsync`. The base class exposes `InputType` and `OutputType` without reflection. It also implements `IValueTaskTransform<TIn, TOut>` for the allocation-free fast path.

### IStreamTransformNode<TIn, TOut>

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

Use when you need to process the entire stream as a whole (e.g., sorting, batching).

### ISinkNode<TIn>

```csharp
public interface ISinkNode<TIn> : INode
{
    Task ConsumeAsync(IDataStream<TIn> input, PipelineContext context, CancellationToken cancellationToken);
}
```

Base class: `SinkNode<TIn>` — extend this, implement `ConsumeAsync`.

### IJoinNode

```csharp
public interface IJoinNode : INode, INodeTypeMetadata
{
    // Base interface — see KeyedJoinNode, TimeWindowedJoinNode, etc.
}
```

Joins combine two input streams of types `TIn1` and `TIn2` into an output stream of type `TOut`. Base class: `BaseJoinNode`.

### IAggregateNode

```csharp
public interface IAggregateNode : INode
{
    // Base interface — see AggregateNode<TIn, TKey, TAccumulate, TResult>
}
```

## Common Built-in Node Classes

| Node Class | Kind | Purpose |
|---|---|---|
| `LambdaNodes.SourceLambdaNode<TOut>` | Source | Inline source from delegate |
| `LambdaNodes.TransformLambdaNode<TIn, TOut>` | Transform | Inline transform from delegate |
| `LambdaNodes.SinkLambdaNode<TIn>` | Sink | Inline sink from delegate |
| `KeyedJoinNode<TKey, TIn1, TIn2, TOut>` | Join | Key-based equi-join (TKey is first type param) |
| `TimeWindowedJoinNode<TKey, TIn1, TIn2, TOut>` | Join | Time-windowed join (TKey is first type param) |
| `AggregateNode<TIn, TKey, TResult>` | Aggregate | Grouped aggregation (TAccumulate = TResult) |
| `AdvancedAggregateNode<TIn, TKey, TAccumulate, TResult>` | Aggregate | Accumulator ≠ result type |
| `InMemoryLookupNode<TIn, TKey, TValue, TOut>` | Lookup | In-memory key/value enrichment (internal — use `AddInMemoryLookup`) |
| `LookupNode<TIn, TKey, TValue, TOut>` | Lookup | Custom lookup transform |
| `RouteNode<T>` | Route | Conditional routing |
| `BranchNode<T>` | Branch | Side-effect branching |
| `TapNode<T>` | Tap | Side-channel monitoring |
| `BatchingNode<T>` | Batch | Groups items (outputs `IReadOnlyCollection<T>`) |
| `UnbatchingNode<T>` | Batch | Flattens `IEnumerable<T>` back to individual items |
| `ReadOnlyCollectionUnbatchingNode<T>` | Batch | Flattens `IReadOnlyCollection<T>` back to individual items |
| `CustomMergeNode<T>` | Transform | Custom stream merging |
