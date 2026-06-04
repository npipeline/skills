# Pipeline Composition Reference

**Package:** `NPipeline.Extensions.Composition`

Embed entire pipelines as transform nodes within larger pipelines. This enables reuse of validated sub-pipelines.

## CompositeTransformNode<TIn, TOut, TDefinition>

Wraps a sub-pipeline definition as a transform node:

```csharp
public class OrderEnrichmentPipeline : IPipelineDefinition
{
    public void Define(PipelineBuilder builder, PipelineContext context)
    {
        var input = builder.AddCompositeInput<Order>();
        var validate = builder.AddTransform<ValidateOrder, Order, Order>();
        var enrich = builder.AddTransform<EnrichOrder, Order, EnrichedOrder>();
        var output = builder.AddCompositeOutput<EnrichedOrder>();

        builder.Connect(input, validate);
        builder.Connect(validate, enrich);
        builder.Connect(enrich, output);
    }
}
```

Key nodes:
- `builder.AddCompositeInput<T>()` — Marks the input boundary
- `builder.AddCompositeOutput<T>()` — Marks the output boundary

## Using Composites in Parent Pipelines

```csharp
// The parent pipeline uses the sub-pipeline as a transform
var source = builder.AddSource<CsvSource, Order>("read");
var enrichment = builder.AddComposite<Order, EnrichedOrder, OrderEnrichmentPipeline>("enrich");
var sink = builder.AddSink<DatabaseSink, EnrichedOrder>("save");

builder.Connect(source, enrichment);
builder.Connect(enrichment, sink);
```

The child pipeline is instantiated fresh for each item (by default, one sub-pipeline run per parent item). At most a parent item takes ~2-3µs of overhead to enter the sub-pipeline.

## CompositeContextConfiguration

Controls what context is inherited from parent to child pipeline:

```csharp
// Inherit everything (preset)
builder.AddComposite<Order, EnrichedOrder, MySubPipeline>(
    CompositeContextConfiguration.InheritAll);

// Custom inheritance
builder.AddComposite<Order, EnrichedOrder, MySubPipeline>(
    contextConfiguration: cfg with
    {
        InheritParentParameters = true,
        InheritParentItems = true,
        InheritParentProperties = true,
        InheritRunIdentity = true,
        InheritLineageSink = true,
        InheritExecutionObserver = true,
        InheritDeadLetterDecorator = true
    });
```

| Configuration Property | Default | Effect |
|---|---|---|
| `InheritParentParameters` | false | Copy Parameters dictionary |
| `InheritParentItems` | false | Share Items dictionary |
| `InheritParentProperties` | false | Share Properties dictionary |
| `InheritRunIdentity` | true | Share pipeline run identity (IDs, name, start time) |
| `InheritLineageSink` | true | Share lineage sink |
| `InheritExecutionObserver` | true | Share execution observer |
| `InheritDeadLetterDecorator` | true | Share dead-letter sink |

## Service Provider Isolation

Composites can optionally use their own `IServiceProvider` for node resolution:

```csharp
builder.AddComposite<Order, EnrichedOrder, MySubPipeline>(
    serviceProvider: subSp,
    contextConfig: CompositeContextConfiguration.InheritAll);
```

When no service provider is specified, the child pipeline uses the same DI container as the parent.

## Naming Convention

Composite nodes use `::` as a namespace separator between parent and child node IDs:

```
parent-pipeline::enrich/validate
```

This appears in logs, lineage trails, and graph visualizations.

## Composition with DI

Register both parent and child pipelines:

```csharp
services.AddNPipeline(builder => builder
    .AddPipeline<MainPipeline>()
    .AddPipeline<OrderEnrichmentPipeline>());
```

Both pipelines participate in assembly scanning.

## Batching and Unbatching

Batching groups individual items; unbatching flattens groups back:

```csharp
var source = builder.AddSource<OrderSource, Order>("source");
var batcher = builder.AddBatcher<Order>("batch", batchSize: 100, timespan: TimeSpan.FromSeconds(5));
var sink = builder.AddSink<BatchSink, IReadOnlyCollection<Order>>("batch-sink");
var unbatch = builder.AddUnbatcher<Order>("unbatch");
var itemSink = builder.AddSink<ItemSink, Order>("item-sink");

// Batch path: processes up to 100 orders (or 5s worth) at a time
builder.Connect(source, batcher);
builder.Connect(batcher, sink);

// Unbatch path: flattens batches back to individual items
builder.Connect(batcher, unbatch);
builder.Connect(unbatch, itemSink);
```

`AddBatcher` requires a `name`, `batchSize`, and `timespan`. Items are emitted when either the batch size or time window is reached. The batcher outputs `IReadOnlyCollection<T>`.

Use `AddUnbatcher<T>` to flatten `IEnumerable<T>` back to individual items, or `AddReadOnlyCollectionUnbatcher<T>` for `IReadOnlyCollection<T>` batches.
