---
name: npipeline-data-flow
description: Use when the user wants to implement advanced data flow patterns in NPipeline. Covers branching (fan-out), routing with RouteNode, joins (keyed, one-to-one, and time-windowed), aggregation with windows, batching/unbatching, taps, lookups, and pipeline composition (sub-pipelines as transforms). Use when user mentions "branch", "route", "join", "aggregate", "batch", "window", "compose pipelines", "tap", "lookup", or "merge".
npipelineVersion: "0.67.0"
lastVerified: "2026-09-29"
---

# NPipeline Data Flow

This skill covers advanced data flow patterns beyond simple linear pipelines: branching, routing, joining, aggregation, batching, taps, lookups, and pipeline composition.

## Workflow

When the user needs complex data flow, identify the pattern they need, then guide them through the implementation using the relevant reference file.

### Branching (Fan-out)

Connect a single source to multiple downstream targets. Each target receives the same data independently:

```csharp
var source = builder.AddSource<MySource, Order>("src");
var validate = builder.AddTransform<ValidateOrder, Order, Order>("validate");
var log = builder.AddSink<LoggingSink, Order>("log");

builder.Connect(source, validate);  // Main flow
builder.Connect(source, log);       // Side logging
```

### Taps

Side-channel sinks that receive copies without affecting the main flow. `AddTap` takes the sink instance (or a factory) and returns a handle to connect:

```csharp
var metricsSink = new MetricsSinkNode();
var tap = builder.AddTap<Order>(metricsSink);
builder.Connect(source, tap);
```

### Branches

Side-effect handlers attached to a stream. `AddBranch` takes one or more `Func<T, Task>` handlers and returns a handle to connect:

```csharp
var branch = builder.AddBranch<Order>(async o => await LogAsync(o));
builder.Connect(source, branch);
```

### Routing (Conditional Fan-out)

Route items to different destinations based on predicates:

```csharp
var route = builder.AddRoute<Order>();
builder.ConnectWhen(route, priorityHandler, o => o.Amount > 1000);
builder.ConnectWhen(route, regularHandler, o => o.Amount <= 1000);
builder.ConnectOtherwise(route, fallbackHandler);  // Unmatched items
```

See `references/routing-branching.md` for match modes, unmatched-item behavior, and branch details.

### Joins

Combine two streams into one:

- **Keyed Join** — Match items by key across two streams (many-to-many or one-to-one)
- **Time-Windowed Join** — Match items within a time window

### Aggregation

Group items and compute results over tumbling or sliding windows.

### Batching

Group items into batches for bulk operations:

```csharp
var batcher = builder.AddBatcher<Order>("batch", batchSize: 100, timespan: TimeSpan.FromSeconds(5));
// ... process batches (output is IReadOnlyCollection<Order>) ...
var unbatch = builder.AddReadOnlyCollectionUnbatcher<Order>("unbatch");
```

### Pipeline Composition

Embed a whole pipeline as a transform node within another pipeline:

```csharp
var enrichment = builder.AddComposite<Order, EnrichedOrder, OrderEnrichmentPipeline>("enrich");
```

See `references/joins-aggregation.md` for join types, aggregation API, and window strategies.
See `references/composition.md` for pipeline composition and context inheritance.

> [!IMPORTANT]
> Join nodes are defined by overriding `CreateOutput(TIn1, TIn2)` and marking keys with `[KeySelector(typeof(T), nameof(...))]` on the class. There is no `JoinAsync` override. Aggregates require a base constructor taking `AggregateNodeConfiguration<TIn>`, and `AggregateNode.GetResult` is `sealed`.
