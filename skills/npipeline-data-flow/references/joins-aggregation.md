# Joins and Aggregation Reference

## KeyedJoinNode<TKey, TIn1, TIn2, TOut>

Equi-join two streams by a key. Items from both streams are matched when their keys are equal. Note: `TKey` is the first type parameter.

### Defining a Keyed Join

```csharp
public class OrderCustomerJoin : KeyedJoinNode<int, Order, Customer, EnrichedOrder>
{
    [KeySelector(typeof(Order))]
    public int SelectOrderKey(Order order) => order.CustomerId;

    [KeySelector(typeof(Customer))]
    public int SelectCustomerKey(Customer c) => c.Id;

    public override Task<EnrichedOrder> JoinAsync(
        Order order, Customer customer, PipelineContext ctx, CancellationToken ct)
    {
        return Task.FromResult(new EnrichedOrder(order, customer));
    }
}
```

The `[KeySelector(Type)]` attribute marks which method extracts the key for each input type. Key type must implement `notnull` (value types, strings).

### Registering a Keyed Join

```csharp
var joinHandle = builder.AddJoin<OrderCustomerJoin, Order, Customer, EnrichedOrder>("join");
builder.Connect(orderSource, joinHandle);   // TIn1 = Order
builder.Connect(customerSource, joinHandle); // TIn2 = Customer
// joinHandle output: EnrichedOrder
```

## TimeWindowedJoinNode<TKey, TIn1, TIn2, TOut>

Matches items that occur within a configured time window of each other. Note: `TKey` is the first type parameter.

```csharp
public class OrderEventJoin : TimeWindowedJoinNode<int, Order, Shipment, EnrichedOrder>
{
    protected override TimeSpan WindowDuration => TimeSpan.FromHours(1);
    protected override int SelectLeftKey(Order o) => o.Id;
    protected override int SelectRightKey(Shipment s) => s.OrderId;
    protected override DateTime GetTimestamp1(Order o) => o.CreatedAt;
    protected override DateTime GetTimestamp2(Shipment s) => s.ShippedAt;

    public override Task<EnrichedOrder> JoinAsync(
        Order order, Shipment shipment, PipelineContext ctx, CancellationToken ct)
    {
        return Task.FromResult(new EnrichedOrder(order, shipment));
    }
}
```

## AggregateNode<TIn, TKey, TResult>

Groups items by key within time windows and computes aggregate results. When the accumulator and result types are the same, use the simplified `AggregateNode<TIn, TKey, TResult>` (which inherits from `AdvancedAggregateNode<TIn, TKey, TResult, TResult>`).

### Simple Aggregation (TAccumulate == TResult)

```csharp
public class RevenueAggregator : AggregateNode<Order, string, decimal>
{
    public override string GetKey(Order order) => order.ProductCategory;
    public override decimal CreateAccumulator() => 0m;
    public override decimal Accumulate(decimal acc, Order order) => acc + order.Amount;
    public override decimal GetResult(decimal acc) => acc;
}
```

### Advanced Aggregation (TAccumulate != TResult)

Use `AdvancedAggregateNode<TIn, TKey, TAccumulate, TResult>` when accumulator and result types differ:

```csharp
public class StatsAggregator : AdvancedAggregateNode<Order, string, OrderStats, OrderSummary>
{
    public override string GetKey(Order o) => o.Category;
    public override OrderStats CreateAccumulator() => new();
    public override OrderStats Accumulate(OrderStats acc, Order o) => acc.Add(o);
    public override OrderSummary GetResult(OrderStats acc) => acc.ToSummary();
}
```

## Window Configuration

Windows are configured via `AggregateNodeConfiguration<TIn>`:

```csharp
var config = new AggregateNodeConfiguration<Order>
{
    WindowAssigner = new TumblingWindowAssigner(TimeSpan.FromMinutes(5)),
    TimestampExtractor = o => o.CreatedAt,
    MaxOutOfOrderness = TimeSpan.FromSeconds(30),
    WatermarkInterval = TimeSpan.FromSeconds(10)
};
```

### Window Types

| Type | Behavior | Use Case |
|---|---|---|
| **Tumbling** | Fixed-size, non-overlapping | "Sales per hour" |
| **Sliding** | Overlapping windows | "Running 30-min average" |

### Timestamps and Watermarks

- **Event time** — Timestamp from the data itself (specified via `TimestampExtractor`)
- **Processing time** — When NPipeline received the item
- **Watermarks** — Progress indicators for event-time processing; emitted at `WatermarkInterval`
- **Out-of-orderness** — Late-arriving items are tolerated up to `MaxOutOfOrderness`

## Self-Joins

Join a stream with itself by connecting the same source to both join inputs:

```csharp
var orders = builder.AddSource<OrderSource, Order>("orders");
var join = builder.AddJoin<SelfJoinNode, Order, Order, OrderPair>("self");
builder.Connect(orders, join);  // TIn1 and TIn2 from same source
```

## In-Memory Lookups

For enrichment without formal joins:

```csharp
builder.AddInMemoryLookup<Order, int, Customer, EnrichedOrder>(
    "customer-lookup",
    customers,
    keyExtractor: o => o.CustomerId,
    outputCreator: (o, c) => new EnrichedOrder(o, c));
```

## Lambda-Based Joins and Aggregation

```csharp
builder.AddJoin((Order o, Customer c) => new EnrichedOrder(o, c));
builder.AddAggregate<Order, string, decimal>(
    keySelector: o => o.Category,
    createAccumulator: () => 0m,
    add: (acc, o) => acc + o.Amount,
    getResult: acc => acc
);
```
