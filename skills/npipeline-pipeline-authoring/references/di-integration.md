# Dependency Injection Integration

`NPipeline.Extensions.DependencyInjection` integrates pipelines with `Microsoft.Extensions.DependencyInjection`.

## NuGet Package

```
dotnet add package NPipeline.Extensions.DependencyInjection
```

## Registration

### Assembly Scanning (Recommended)

Discovers and registers all `INode`, `IPipelineDefinition`, `IResiliencePolicy`, `IDeadLetterSink`, `ILineageSink`, `IPipelineLineageSink`, and `IPipelineLineageSinkProvider` implementations:

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
| `ScanAssemblies(params Assembly[])` | Assembly scan for the above types |

Most `AddX<T>` methods have an overload taking a `ServiceLifetime`; all default to `Transient`.

### Service Lifetimes

| Service | Lifetime |
|---|---|
| `PipelineBuilder` | Transient |
| `IPipelineFactory` / `PipelineFactory` | Singleton |
| `INodeFactory` / `DiContainerNodeFactory` | Scoped |
| `IPipelineRunner` / `PipelineRunner` | Scoped |
| `INodeExecutor` | Scoped |
| `ITopologyService` | Scoped |
| `IErrorHandlingService` | Transient |
| `IObservabilitySurface` | Singleton (null surface unless observability is registered) |

## Running Pipelines from DI

```csharp
var provider = services.BuildServiceProvider();
await provider.RunPipelineAsync<MyPipeline>();
```

`RunPipelineAsync` creates a scope, resolves the runner, creates a context wired to the container, and disposes both.

## Creating a Context Yourself

When you resolve the runner and run it directly, create the context through the container so it receives the container's services (logger factory, tracer, observability collector, and every registered `IExecutionObserver`):

```csharp
await using var scope = serviceProvider.CreateAsyncScope();
var runner = scope.ServiceProvider.GetRequiredService<IPipelineRunner>();
await using var context = scope.ServiceProvider.CreatePipelineContext(
    PipelineContextConfiguration.WithCancellation(cancellationToken));
await runner.RunAsync<MyPipeline>(context);
```

> [!WARNING]
> A context created with `new PipelineContext()` or `PipelineContext.CreateDefault()` is not wired to the container. Lineage reports, metrics, and NPipeline's own logging are silently lost. Use the context the provider of the run's scope creates.

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

    public override async ValueTask<EnrichedOrder> TransformAsync(
        Order item, PipelineContext context, CancellationToken ct)
    {
        var enriched = await _service.EnrichAsync(item, ct);
        _logger.LogInformation("Enriched order {Id}", item.Id);
        return enriched;
    }
}
```

The `DiContainerNodeFactory` uses compiled expression trees for fast constructor invocation, with `ActivatorUtilities` as a fallback.

## Ownership and Disposal

The run disposes only the instances it owns. Instances resolved from the container are left to the container; instances created from a configured type are disposed at the end of each run, not with the context.
