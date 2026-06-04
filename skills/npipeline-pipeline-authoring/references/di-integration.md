# Dependency Injection Integration

`NPipeline.Extensions.DependencyInjection` integrates pipelines with `Microsoft.Extensions.DependencyInjection`.

## NuGet Package

```
dotnet add package NPipeline.Extensions.DependencyInjection
```

## Registration

### Assembly Scanning (Recommended)

Discovers and registers all `INode`, `IPipelineDefinition`, `IResiliencePolicy`, `IDeadLetterSink`, `ILineageSink`, and related implementations:

```csharp
services.AddNPipeline(typeof(MyPipeline).Assembly);
```

### Fluent Registration

```csharp
services.AddNPipeline(builder => builder
    .AddNode<MyTransform>()
    .AddNode<MySink>()
    .AddPipeline<MyPipeline>()
    .AddResiliencePolicy<MyPolicy>()
    .AddDeadLetterSink<MyDeadLetterSink>());
```

### NPipelineServiceBuilder Methods

| Method | Purpose |
|---|---|
| `AddNode<T>()` | Register a node for DI instantiation |
| `AddPipeline<T>()` | Register a pipeline definition |
| `AddResiliencePolicy<T>()` | Register a resilience policy |
| `AddDeadLetterSink<T>()` | Register a dead-letter sink |
| `AddLineageSink<T>()` | Register an item-level lineage sink |
| `AddPipelineLineageSink<T>()` | Register a pipeline-level lineage sink |
| `AddLineageSinkProvider<T>()` | Register a lineage sink provider |
| `ScanAssemblies(Assembly[])` | Assembly scan for the above types |

### Service Lifetimes

| Service | Lifetime |
|---|---|
| `PipelineBuilder` | Transient |
| `IPipelineFactory` / `PipelineFactory` | Singleton |
| `INodeFactory` / `DiContainerNodeFactory` | Scoped |
| `IPipelineRunner` / `PipelineRunner` | Scoped |
| `INodeExecutor` | Scoped |
| `ITopologyService` | Scoped |
| `IErrorHandlingService` | Scoped |

## Running Pipelines from DI

```csharp
var provider = services.BuildServiceProvider();
await provider.RunPipelineAsync<MyPipeline>();
```

## Constructor Injection in Nodes

Nodes instantiated via DI support constructor injection:

```csharp
public class MyTransform : TransformNode<Order, EnrichedOrder>
{
    private readonly IOrderService _service;
    private readonly ILogger<MyTransform> _logger;

    public MyTransform(IOrderService service, ILogger<MyTransform> logger)
    {
        _service = service;
        _logger = logger;
    }

    public override async Task<EnrichedOrder> TransformAsync(
        Order item, PipelineContext context, CancellationToken ct)
    {
        var enriched = await _service.EnrichAsync(item, ct);
        _logger.LogInformation("Enriched order {Id}", item.Id);
        return enriched;
    }
}
```

The `DiContainerNodeFactory` uses compiled expression trees for fast constructor invocation, with `ActivatorUtilities` as a fallback.
