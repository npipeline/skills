---
name: npipeline-data-flow
description: Use when the user wants to implement advanced data flow patterns in NPipeline. Covers branching (fan-out), routing with RouteNode, joins (keyed and time-windowed), aggregation with windows, batching/unbatching, taps, lookups, and pipeline composition (sub-pipelines as transforms). Use when user mentions "branch", "route", "join", "aggregate", "batch", "window", "compose pipelines", "tap", "lookup", or "merge".
npipelineVersion: "0.52.0"
lastVerified: "2026-06-04"
---

# NPipeline Data Flow

This skill covers advanced data flow patterns beyond simple linear pipelines: branching, routing, joining, aggregation, batching, taps, lookups, and pipeline composition.

## Workflow

When the user needs complex data flow, identify the pattern they need from the list below, then guide them through the implementation using the relevant reference file.

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

Side-channel sinks that receive copies without affecting the main flow:

```csharp
var metricsSink = new MetricsSinkNode();
builder.AddTap(metricsSink);   // metricsSink receives a copy of every item
```

### Branches

Side-effect handlers attached to a stream:

```csharp
var branchHandle = builder.AddBranch<Order>(async o => await LogAsync(o));
builder.Connect(sourceHandle, branchHandle);
```

### Routing (Conditional Fan-out)

Route items to different destinations based on predicates:

```csharp
var routeHandle = builder.AddRoute<Order>();
builder.ConnectWhen(routeHandle, priorityHandler, o => o.Amount > 1000);
builder.ConnectWhen(routeHandle, regularHandler, o => o.Amount <= 1000);
builder.ConnectOtherwise(routeHandle, fallbackHandler);  // Unmatched items
```

See `references/routing-branching.md` for full routing API with match modes, unmatched item behavior, and branch node details.

### Joins

Combine two streams into one:

- **Keyed Join** — Match items by key across two streams
- **Time-Windowed Join** — Match items within a time window

### Aggregation

Group items and compute results over windows:

- **Tumbling Windows** — Non-overlapping, fixed-size windows
- **Sliding Windows** — Overlapping windows

### Batching

Group items into batches for bulk operations:

```csharp
builder.AddBatcher<Order>("batch", batchSize: 100, timespan: TimeSpan.FromSeconds(5));
// ... process batches (output is IReadOnlyCollection<Order>) ...
builder.AddUnbatcher<Order>("unbatch");
```

### Pipeline Composition

Embed a whole pipeline as a transform node within another pipeline:

```csharp
builder.AddComposite<Order, EnrichedOrder, OrderEnrichmentPipeline>("enrich");
```

See `references/joins-aggregation.md` for join types, aggregation API, and window strategies.
See `references/composition.md` for pipeline composition and context inheritance.
