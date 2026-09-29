---
name: npipeline-node-development
description: Use when the user wants to write custom NPipeline node classes. Covers extending SourceNode, TransformNode, SinkNode, implementing stream transforms, the ValueTask-native transform API, resource disposal, constructor injection with DI, and node metadata attributes. Use when user mentions "write a custom node", "create a transform", "SourceNode", "TransformNode", "SinkNode", "custom source", "custom sink", or "implement a node".
npipelineVersion: "0.67.0"
lastVerified: "2026-09-29"
---

# NPipeline Node Development

This skill covers authoring custom node classes: sources that produce data, transforms that process it, and sinks that consume it.

## Workflow

When the user wants to write a custom node, determine which base class they need, then guide them through implementation following the patterns below.

### Phase 1: Choose the Right Base Class

| If the node... | Use base class | Implement |
|---|---|---|
| Produces data from scratch | `SourceNode<TOut>` | `OpenStream` |
| Takes one item, returns one item | `TransformNode<TIn, TOut>` | `TransformAsync` |
| Processes the entire stream (sort, batch, filter) | `IStreamTransformNode<TIn, TOut>` | `TransformAsync` |
| Consumes items (terminal) | `SinkNode<TIn>` | `ConsumeAsync` |
| Enriches from a lookup table | `LookupNode<TIn, TKey, TValue, TOut>` | `ExtractKey`, `LookupAsync`, `CreateOutput` |
| Custom merge logic for several upstreams | `CustomMergeNode<TIn>` | `MergeAsync` |

Consult `references/base-classes.md` for detailed base class APIs.
Consult `references/patterns.md` for disposal, DI, and metadata attribute patterns.

> [!IMPORTANT]
> `TransformAsync` returns `ValueTask<TOut>`, not `Task<TOut>`. There is no separate `IValueTaskTransform` fast path to opt into: returning `ValueTask` directly is the fast path, and a transform that completes synchronously (`ValueTask.FromResult(...)`) allocates nothing per item.

### Phase 2: Implement the Node

Always extend the base class rather than directly implementing the interface. This gives you built-in type metadata and the covariant/contravariant interface wiring the runtime relies on.

### Phase 3: Register the Node

```csharp
// In the pipeline definition
builder.AddSource<MySource, MyData>("source-name");
builder.AddTransform<MyTransform, MyData, MyOutput>("transform-name");
builder.AddSink<MySink, MyOutput>("sink-name");
```

For DI scenarios, ensure the node has a public constructor and optionally register it:

```csharp
services.AddNPipeline(builder => builder.AddNode<MyTransform>());
```

## Key Conventions

- Always forward `CancellationToken` to every async operation.
- Use `.WithCancellation(ct)` on `IAsyncEnumerable<T>` enumerations.
- A node holds no execution strategy of its own: how the node runs is configured on the graph with `WithExecutionStrategy`. A node type with an inherent default strategy implements `IExecutionStrategyProvider`.
- A node that owns resources implements `IAsyncDisposable` or `IDisposable` itself. The runtime checks for the interface and disposes the instance at the end of the run that created it. There is no base implementation to call.
- Provide a public parameterless constructor, or register the node with DI for constructor injection.
