# PipelineContext and Configuration Reference

## PipelineContext

`PipelineContext` is the execution-scoped bag of everything needed during a single pipeline run. It is constructed from `PipelineContextConfiguration`.

### PipelineContextConfiguration

```csharp
public sealed record PipelineContextConfiguration(
    IDictionary<string, object>? Parameters = null,
    IDictionary<string, object>? Items = null,
    IDictionary<string, object>? Properties = null,
    IErrorHandlerFactory? ErrorHandlerFactory = null,
    IResiliencePolicy? ResiliencePolicy = null,
    IDeadLetterSink? DeadLetterSink = null,
    ILoggerFactory? LoggerFactory = null,
    IPipelineTracer? Tracer = null,
    IObservabilityFactory? ObservabilityFactory = null,
    ILineageFactory? LineageFactory = null,
    PipelineOptimizationProfile OptimizationProfile = PipelineOptimizationProfile.Default,
    CancellationToken CancellationToken = default)
{
    public Guid RunId { get; init; }
    public string? PipelineName { get; init; }

    static PipelineContextConfiguration Default { get; }
    static PipelineContextConfiguration WithFactories(IErrorHandlerFactory?, ILineageFactory?, IObservabilityFactory?)
    static PipelineContextConfiguration WithLogging(ILoggerFactory loggerFactory)
    static PipelineContextConfiguration WithParameters(IDictionary<string, object> parameters)
    static PipelineContextConfiguration WithCancellation(CancellationToken cancellationToken)
    static PipelineContextConfiguration WithObservability(ILoggerFactory?, IPipelineTracer?)
    static PipelineContextConfiguration WithErrorHandling(IDeadLetterSink?)
    static PipelineContextConfiguration WithResilience(IResiliencePolicy resiliencePolicy)
}
```

`RunId` lets a caller correlate the run with an identifier their own system already holds. `PipelineName` overrides the name reported by the pipeline definition.

> [!NOTE]
> `INodeFactory` and `IPipelineFactory` are not configuration properties. The node factory comes from the runner.

### Context Dictionaries

| Dictionary | Thread Safety | Purpose |
|---|---|---|
| `Parameters` | Read-only | Runtime parameters set once before execution |
| `Items` | Profile-dependent | Mutable key/value store for pipeline-wide state |
| `Properties` | Profile-dependent | Extension point for framework-level metadata |

**Default profile**: thread-safe `ConcurrentDictionary` for `Items` and `Properties`. A caller-supplied `Parameters` dictionary that is not already concurrent is copied; non-concurrent `Items` and `Properties` are wrapped in a synchronized wrapper.
**HighThroughput profile**: pooled `Dictionary` (no locking; the user must ensure single-threaded access).

### Context Composition

`PipelineContext` composes five sub-contexts:

| Sub-Context | Properties |
|---|---|
| `RunIdentity` | `PipelineId`, `RunId`, `PipelineName`, `PipelineStartTimeUtc` |
| `ExecutionConfiguration` | Resolved resilience options per node, parallel execution flag |
| `Observability` | `LoggerFactory`, `Tracer`, `ObservabilityFactory`, `ExecutionObserver` |
| `NodeEnvironment` | Node execution scope registry, preconfigured nodes |
| `Lineage` | Lineage factory, lineage sink, pipeline lineage sink, lineage collector |

## PipelineRunner

### Creating a Runner

```csharp
// Simplest: all defaults
var runner = PipelineRunner.Create();

// With custom dependencies
var runner = new PipelineRunnerBuilder()
    .WithNodeFactory(myNodeFactory)
    .WithObservabilitySurface(mySurface)
    // ... other overrides
    .Build();
```

### Running a Pipeline

```csharp
// No context (default context created for you)
await runner.RunAsync<MyPipeline>();

// With a context
var context = new PipelineContext(new PipelineContextConfiguration(
    Parameters: new Dictionary<string, object> { ["inputPath"] = "/data/file.csv" }
));
await runner.RunAsync<MyPipeline>(context);

// With an already-constructed definition (constructor-injected parameters)
var definition = new MyPipeline("/data/file.csv");
await runner.RunAsync(definition, context, cts.Token);

// From IServiceProvider
await serviceProvider.RunPipelineAsync<MyPipeline>();
```

> [!TIP]
> Under DI, prefer `serviceProvider.CreatePipelineContext(...)` (or `RunPipelineAsync`) over `new PipelineContext()`. A bare context is not wired to the container's logger factory, tracer, observability collector, or execution observers. See `references/di-integration.md`.

## Optimization Profiles

| Profile | Dictionary Type | Auto-Configured Retry | Analyzer Rules | Use Case |
|---|---|---|---|---|
| `Default` | `ConcurrentDictionary` | Item retry: 3 retries (exp. backoff + full jitter) | Suppressed NP9103-9107 | Prototyping, low-medium throughput |
| `HighThroughput` | Pooled `Dictionary` | None | All active | Millions of items/second |

```csharp
builder.WithOptimizationProfile(PipelineOptimizationProfile.HighThroughput);
```

The profile sets the resilience options that `WithResilience(...)` starts from (see `PipelineResilienceOptions.ForProfile`).

## Pipeline Execution Lifecycle

1. **Build** — `PipelineBuilder.Build()` validates the graph and freezes collections
2. **Setup** — the runtime binder resolves stream contracts, the node factory creates instances, and the registration planner compiles execution plans
3. **Execution** — the orchestrator walks the graph and processes data through each node
4. **Cleanup** — the run disposes the instances it owns; `PipelineContext.DisposeAsync()` disposes registered async disposables

## Dependency Injection in the Context

```csharp
// Under DI, create the context from the run's scope so it gets the container's services
await using var scope = serviceProvider.CreateAsyncScope();
var runner = scope.ServiceProvider.GetRequiredService<IPipelineRunner>();
await using var context = scope.ServiceProvider.CreatePipelineContext(
    PipelineContextConfiguration.WithCancellation(cancellationToken));
await runner.RunAsync<MyPipeline>(context);
```
