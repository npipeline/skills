# Joins and Aggregation Reference

## KeyedJoinNode<TKey, TIn1, TIn2, TOut>

Equi-join two streams by a key. Items from both streams are matched when their keys are equal. `TKey` is the first type parameter.

### Defining a Keyed Join

Mark each input type's key with a class-level `[KeySelector]` attribute, and override `CreateOutput`:

```csharp
using NPipeline.Attributes.Nodes;
using NPipeline.Nodes;

[KeySelector(typeof(Order), nameof(Order.CustomerId))]
[KeySelector(typeof(Customer), nameof(Customer.CustomerId))]
public class OrderCustomerJoin : KeyedJoinNode<int, Order, Customer, EnrichedOrder>
{
    public override EnrichedOrder CreateOutput(Order order, Customer customer)
        => new(order, customer);

    // For outer joins, override the unmatched-side output too:
    public override EnrichedOrder CreateOutputFromLeft(Order order)
        => new(order, Customer: null!);
}
```

> [!IMPORTANT]
> Joins are defined by overriding `CreateOutput(TIn1, TIn2)`, not `JoinAsync`. The `[KeySelector]` attribute names the property (or properties, for a composite `ValueTuple` key) that supplies the key for each input type.

Input types must be distinct: a join whose input types are assignable to each other is rejected when the node is constructed. Use `AddSelfJoin` to join a stream with itself.

### Registering and Connecting

```csharp
var join = builder.AddJoin<OrderCustomerJoin, Order, Customer, EnrichedOrder>("join");
builder.Connect(orderSource, join);      // TIn1 = Order
builder.Connect(customerSource, join);   // TIn2 = Customer
// join output: EnrichedOrder
```

### Join Type and Memory

`JoinType` (default `Inner`) controls matching; the unmatched-side outputs above are used for `LeftOuter`, `RightOuter`, and `FullOuter`. Both inputs are read concurrently. A null key never matches (as in SQL). `MaxCapacity` bounds how many items each input retains.

### One-to-One Joins

When each key occurs at most once per side, set `Cardinality = JoinCardinality.OneToOne` so items are released as soon as they match:

```csharp
public class OrderPaymentJoin : KeyedJoinNode<string, Order, Payment, PaidOrder>
{
    public OrderPaymentJoin()
    {
        Cardinality = JoinCardinality.OneToOne;
        DuplicateKeyPolicy = DuplicateKeyPolicy.DeadLetter;
        JoinType = JoinType.LeftOuter;
    }
}
```

`DuplicateKeyPolicy` is `Drop` (default), `DeadLetter` (raises `DuplicateJoinKeyException`, NP0426), or `EmitAsUnmatched`. `MaxMatchedKeys` caps the set of remembered matched keys. Setting `DuplicateKeyPolicy` or `MaxMatchedKeys` on a many-to-many join fails the join when it starts (NP0427).

## TimeWindowedJoinNode<TKey, TIn1, TIn2, TOut>

Matches items that occur within a time window of each other. Construct the node with a `WindowAssigner` and optional timestamp extractors:

```csharp
public class TradeSettlementJoin
    : TimeWindowedJoinNode<string, Trade, Settlement, MatchedTrade>
{
    public TradeSettlementJoin() : base(
        WindowAssigner.Tumbling(TimeSpan.FromMinutes(5)),
        timestampExtractor1: trade => trade.ExecutedAt,
        timestampExtractor2: settlement => settlement.SettledAt,
        maxOutOfOrderness: TimeSpan.FromMinutes(2))
    { }

    public override MatchedTrade CreateOutput(Trade trade, Settlement settlement)
        => new(trade, settlement);
}
```

Watermarks follow event time (the timestamp extractor, `ITimestamped`, or arrival time), align on UTC, and follow the slower input. Items arriving after their window closes are dropped and counted in `LateItemsDropped`; an outer join emits such an item at once as unmatched when its side is preserved.

## AggregateNode<TIn, TKey, TResult>

Groups items by key within time windows and computes aggregate results. Extend and pass an `AggregateNodeConfiguration<TIn>` to the base constructor.

### Simple Aggregation (TAccumulate == TResult)

```csharp
public class RevenueAggregator : AggregateNode<Order, string, decimal>
{
    public RevenueAggregator() : base(new AggregateNodeConfiguration<Order>(
        WindowAssigner.Tumbling(TimeSpan.FromMinutes(5))))
    { }

    public override string GetKey(Order order) => order.ProductCategory;
    public override decimal CreateAccumulator() => 0m;
    public override decimal Accumulate(decimal acc, Order order) => acc + order.Amount;
    // GetResult is sealed in this class and returns the accumulator as-is.
}
```

### Advanced Aggregation (TAccumulate != TResult)

Extend `AdvancedAggregateNode<TIn, TKey, TAccumulate, TResult>` and override `GetResult` when accumulator and result types differ:

```csharp
public class StatsAggregator : AdvancedAggregateNode<Order, string, OrderStats, OrderSummary>
{
    public StatsAggregator() : base(new AggregateNodeConfiguration<Order>(
        WindowAssigner.Sliding(TimeSpan.FromMinutes(5), TimeSpan.FromMinutes(1))))
    { }

    public override string GetKey(Order o) => o.Category;
    public override OrderStats CreateAccumulator() => new();
    public override OrderStats Accumulate(OrderStats acc, Order o) => acc.Add(o);
    public override OrderSummary GetResult(OrderStats acc) => acc.ToSummary();
}
```

## Window Configuration

Windows are configured through `AggregateNodeConfiguration<TIn>`:

```csharp
public sealed record AggregateNodeConfiguration<TIn>(
    WindowAssigner WindowAssigner,
    TimestampExtractor<TIn>? TimestampExtractor = null,
    TimeSpan? MaxOutOfOrderness = null);
```

| Property | Default | Description |
|---|---|---|
| `WindowAssigner` | required | Tumbling or sliding window strategy; sizes and slides must be positive |
| `TimestampExtractor` | `null` | Extracts event time; falls back to `ITimestamped` or arrival time |
| `MaxOutOfOrderness` | 5 minutes | How late an item can arrive; must not be negative |

### Window Types

| Type | Factory | Behavior |
|---|---|---|
| **Tumbling** | `WindowAssigner.Tumbling(windowSize)` | Fixed-size, non-overlapping |
| **Sliding** | `WindowAssigner.Sliding(windowSize, slide)` | Overlapping windows |

> [!NOTE]
> `AggregateNodeConfiguration<TIn>` has exactly three properties: `WindowAssigner`, `TimestampExtractor`, and `MaxOutOfOrderness`. The watermark is re-evaluated on every item, so windows close as soon as the data passes them, even in a fast replay.

## Self-Joins

Join a stream with itself using `AddSelfJoin`, which takes two source handles, an output factory, and key selectors:

```csharp
var joined = builder.AddSelfJoin<Event, string, MatchedEvent>(
    leftSource, rightSource, "self-join",
    outputFactory: (e1, e2) => new MatchedEvent(e1, e2),
    leftKeySelector: e => e.CorrelationId);

builder.Connect(joined, sink);
```

`AddSelfJoin` takes optional `cardinality` and `duplicateKeyPolicy` parameters for a one-to-one join.

## In-Memory Lookups

For enrichment without formal joins:

```csharp
builder.AddInMemoryLookup<Order, int, Customer, EnrichedOrder>(
    "customer-lookup",
    lookupData: customers,
    keyExtractor: o => o.CustomerId,
    outputCreator: (o, c) => new EnrichedOrder(o, c));
```

## Lambda-Based Aggregation

For simple cases, use the fluent grouping API rather than a node class:

```csharp
// Tumbling window
var totals = builder.GroupItems<Sale>()
    .ForTemporalCorrectness(
        windowSize: TimeSpan.FromHours(1),
        keySelector: sale => sale.Category,
        initialValue: () => 0m,
        accumulator: (sum, sale) => sum + sale.Amount,
        timestampExtractor: sale => sale.Timestamp);

// Sliding window
var averages = builder.GroupItems<Measurement>()
    .ForRollingWindow(
        windowSize: TimeSpan.FromMinutes(5),
        slideInterval: TimeSpan.FromMinutes(1),
        keySelector: m => m.SensorId,
        initialValue: () => (Sum: 0.0, Count: 0),
        accumulator: (acc, m) => (acc.Sum + m.Value, acc.Count + 1),
        timestampExtractor: m => m.Timestamp);
```
