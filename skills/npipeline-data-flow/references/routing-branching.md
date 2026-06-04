# Routing and Branching Reference

## RouteNode<T>

Conditional fan-out. Items are evaluated against predicates and routed to matching destinations.

### Route Builder API

Routes are added and connected in separate steps:

```csharp
var route = builder.AddRoute<Order>();
builder.ConnectWhen(route, priorityHandler, o => o.Amount > 1000);
builder.ConnectWhen(route, regularHandler, o => o.Amount <= 1000);
builder.ConnectOtherwise(route, deadLetterHandler);
```

Or configure up-front with options:

```csharp
var route = builder.AddRoute<Order>(options => {
    options.When("priority", o => o.Amount > 1000);
    options.When("regular", o => o.Amount <= 1000);
});
builder.ConnectWhen(route, priorityHandler, o => o.Amount > 1000);
builder.ConnectWhen(route, regularHandler, o => o.Amount <= 1000);
```

You can also chain via fluent builder methods:

```csharp
builder.AddRoute<Order>()
    .Let(route => builder
        .ConnectWhen(route, priorityHandler, o => o.Amount > 1000)
        .ConnectOtherwise(route, otherHandler));
```

### Match Modes

| Mode | Behavior |
|---|---|
| `FirstMatch` (default) | Item goes to the first matching route only |
| `AllMatches` | Item is sent to every matching route |

Configure via `RouteOptions<T>`:

### Unmatched Item Behavior

Configure via `RouteOptions<T>` when adding the route, or via `builder.ConfigureRoute(route, options => ...)`: discard unmatched, throw, or route otherwise.

## BranchNode<T>

Attaches side-effect handlers to a stream — items flow through to downstream nodes while the branch runs its handler:

```csharp
var branchHandle = builder.AddBranch<Order>(async o => await LogAsync(o));
builder.Connect(source, branchHandle);
```

To add multiple handlers at once:

```csharp
var branchHandle = builder.AddBranch<Order>(new Func<Order, Task>[] {
    async o => await LogAsync(o),
    async o => await MetricsAsync(o)
});
builder.Connect(source, branchHandle);
```

## TapNode<T>

Side-channel monitoring — copies items to a sink without affecting the main flow:

```csharp
var metricsSink = builder.AddSink<MetricsSink, Order>("metrics");
builder.AddTap(metricsSink);
```

Taps are useful for logging, metrics, or auditing without changing pipeline behavior.

## Fan-out (Multiple Connect)

The simplest branching pattern — connect one source to multiple targets:

```csharp
builder.Connect(source, validate);
builder.Connect(source, enrich);
builder.Connect(source, log);
```

Each target receives every item. Processing is independent across branches.

## Fan-in / Merge

Combine multiple streams interleaved (default is interleaved merge). For custom merge strategies, implement `ICustomMergeNode`:

```csharp
builder.AddCustomMerge<Order>(merge => { /* custom logic */ });
```

## Lookups

Enrich items by resolving values from a key-value store using `builder.AddInMemoryLookup()`:

```csharp
builder.AddInMemoryLookup<Order, int, Customer, EnrichedOrder>(
    "customer-lookup",
    customers,
    keyExtractor: o => o.CustomerId,
    outputCreator: (o, c) => new EnrichedOrder(o, c));
```

For custom lookups, extend `LookupNode<TIn, TKey, TValue, TOut>` and override `ExtractKey`, `LookupAsync`, and `CreateOutput`. Note that `InMemoryLookupNode` is an internal type — always use the `AddInMemoryLookup` builder extension.
