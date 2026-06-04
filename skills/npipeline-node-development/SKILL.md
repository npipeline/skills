---
name: npipeline-node-development
description: Use when the user wants to write custom NPipeline node classes. Covers extending SourceNode, TransformNode, SinkNode, implementing stream transforms, ValueTask fast paths, resource disposal, constructor injection with DI, and node metadata attributes. Use when user mentions "write a custom node", "create a transform", "SourceNode", "TransformNode", "SinkNode", "custom source", "custom sink", or "implement a node".
npipelineVersion: "0.52.0"
lastVerified: "2026-06-04"
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
| Processes the entire stream (sort, batch) | `IStreamTransformNode<TIn, TOut>` | `TransformAsync` |
| Consumes items (terminal) | `SinkNode<TIn>` | `ConsumeAsync` |

Consult `references/base-classes.md` for detailed base class APIs.
Consult `references/patterns.md` for ValueTask fast paths, disposal, and DI patterns.

### Phase 2: Implement the Node

Always extend the base class rather than directly implementing the interface. This gives you built-in type metadata, disposal plumbing, and the `ValueTask` fast path.

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
- Override `DisposeAsync()` if holding resources; always call `base.DisposeAsync()`.
- Provide a public parameterless constructor (or register with DI via constructor injection).
- Override `ExecuteValueTaskAsync` for synchronous transforms to avoid per-item Task allocations.
