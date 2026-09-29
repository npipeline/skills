# Routing and Branching Reference

## RouteNode<T>

Conditional fan-out. Items are evaluated against predicates and routed to matching outputs.

### Route Builder API

Add a route and connect named outputs with predicates:

```csharp
var route = builder.AddRoute<Order>("route-orders");
builder.ConnectWhen(route, priorityHandler, o => o.Amount > 1000, "high-value");
builder.ConnectWhen(route, regularHandler, o => o.Amount <= 1000, "standard");
builder.ConnectOtherwise(route, deadLetterHandler);  // Fallback output
```

You can also add the route with an options callback:

```csharp
var route = builder.AddRoute<Order>(options =>
{
    options.WithMatchMode(RouteMatchMode.AllMatches);
}, "route-orders");
```

The `ConnectWhen` and `ConnectOtherwise` overloads take an optional `sourceOutputName`. When omitted, the output name defaults to the target node's id, or `RouteOutputNames.Otherwise` ("otherwise") for `ConnectOtherwise`.

### Match Modes

`RouteOptions<T>` supports two match modes:

| Mode | Behavior |
|---|---|
| `FirstMatch` (default) | Routes to the first matching rule only |
| `AllMatches` | Routes to every output whose rule matches |

```csharp
builder.ConfigureRoute(route, options => options.WithMatchMode(RouteMatchMode.AllMatches));
```

> [!NOTE]
> `ConnectOtherwise` fires only when no explicit rule matches, regardless of match mode. In `AllMatches` mode, an item that satisfies at least one `ConnectWhen` predicate does not reach the otherwise output.

### Unmatched Item Behavior

If no rule matches and no otherwise output is configured, unmatched items are dropped by default. Change this with `WithNoMatchBehavior`:

```csharp
builder.ConfigureRoute(route, options => options.WithNoMatchBehavior(NoRouteMatchBehavior.Throw));
```

| Option | Behavior |
|---|---|
| `NoRouteMatchBehavior.Drop` (default) | Silently drops unmatched items |
| `NoRouteMatchBehavior.Throw` | Throws to fail fast on routing gaps |

`RouteOptions<T>` also exposes `When(outputName, predicate)` and `Otherwise(outputName?)` for direct configuration. `IRouteOptions` reads a route's configuration without its item type, for tooling.

### Naming Outputs for Tooling

Use stable, semantic output names (`high-value`, `international`, `standard`) so diagrams and tooling render route branches consistently.

### Route Predicates and Lineage

Route predicates always operate on the **payload type** `T`. When item-level lineage is enabled, stream items are wrapped in `LineagePacket<T>` at runtime, but `RuntimePipelineBinder` normalizes `RouteOptions<T>` to `RouteOptions<LineagePacket<T>>` once at bind time. You never write predicates against `LineagePacket<T>`.

## BranchNode<T>

Attaches side-effect handlers to a stream: items flow through to downstream nodes while the branch runs its handler.

```csharp
var branch = builder.AddBranch<Order>(async o => await LogAsync(o), "notify-branch");
builder.Connect(source, branch);
builder.Connect(branch, nextTransform);
```

Add multiple handlers at once (they run in parallel):

```csharp
var branch = builder.AddBranch<Order>(new Func<Order, Task>[]
{
    async o => await LogAsync(o),
    async o => await MetricsAsync(o),
}, "multi-branch");
```

## TapNode<T>

Side-channel monitoring: copies items to a sink without affecting the main flow. `AddTap` returns a pass-through handle you connect on both sides.

```csharp
var source = builder.AddSource<OrderSource, Order>("orders");
var auditTap = builder.AddTap<Order>(() => new AuditSink(logger), "audit");
var transform = builder.AddTransform<ProcessOrder, Order, Result>("process");

builder.Connect(source, auditTap);
builder.Connect(auditTap, transform);  // items continue downstream
```

The pipeline drives the tap's sink once over the whole stream. A bounded channel buffers items between the main flow and the sink. Taps are useful for logging, metrics, or auditing without changing pipeline behavior.

## Fan-out (Multiple Connect)

The simplest branching pattern: connect one source to multiple targets.

```csharp
builder.Connect(source, validate);
builder.Connect(source, enrich);
builder.Connect(source, log);
```

Each target receives every item. A fan-out branch is released once every terminal below it has finished, so a sink that stops early or never reads cannot stall its siblings.

## Fan-in / Merge

Combining several streams into a node with multiple inputs uses an interleaved merge by default. Fan-in merges are bounded (1,024 items by default) and fail as soon as one input fails. Bound a merge with `WithMergeCapacity` / `WithGlobalMergeCapacity`. For custom merge logic, implement `ICustomMergeNode<TIn>` (extend `CustomMergeNode<TIn>`).

## Lookups

Enrich items by resolving values from a key-value store using `builder.AddInMemoryLookup`:

```csharp
var lookup = builder.AddInMemoryLookup<Order, int, Customer, EnrichedOrder>(
    "customer-lookup",
    lookupData: customers,
    keyExtractor: o => o.CustomerId,
    outputCreator: (o, c) => new EnrichedOrder(o, c));

builder.Connect(source, lookup);
```

For custom lookups, extend `LookupNode<TIn, TKey, TValue, TOut>` and override `ExtractKey`, `LookupAsync` (returns `ValueTask<TValue?>`), and `CreateOutput`.
