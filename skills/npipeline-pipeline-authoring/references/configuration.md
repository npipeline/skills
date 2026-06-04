# PipelineContext and Configuration Reference

## PipelineContext

`PipelineContext` is the execution-scoped bag of everything needed during a single pipeline run. It is constructed from `PipelineContextConfiguration`.

### PipelineContextConfiguration

```csharp
public sealed record PipelineContextConfiguration
{
    // Shared dictionaries
    IReadOnlyDictionary<string, object>? Parameters  // Read-only, set once at creation
    IDictionary<string, object>? Items                // Mutable shared state
    IDictionary<string, object>? Properties           // Extension point

    // Cancellation
    CancellationToken CancellationToken

    // Optimization profile (controls thread safety model)
    PipelineOptimizationProfile OptimizationProfile

    // Factory overrides (for DI scenarios)
    INodeFactory? NodeFactory
    IPipelineFactory? PipelineFactory
    IObservabilityFactory? ObservabilityFactory
    ILineageFactory? LineageFactory
    IErrorHandlerFactory? ErrorHandlerFactory

    // Static factory methods
    static PipelineContextConfiguration Default { get; }
    static PipelineContextConfiguration WithParameters(IDictionary<string, object> parameters)
    static PipelineContextConfiguration WithLogging(ILoggerFactory loggerFactory)
    static PipelineContextConfiguration WithCancellation(CancellationToken cancellationToken)
}
```

### Context Dictionaries

| Dictionary | Thread Safety | Purpose |
|---|---|---|
| `Parameters` | Read-only | Runtime parameters set once before execution |
| `Items` | Profile-dependent | Mutable key/value store for pipeline-wide state |
| `Properties` | Profile-dependent | Extension point for framework-level metadata |

**Default profile**: Thread-safe `ConcurrentDictionary` for `Items` and `Properties`.
**HighThroughput profile**: Pooled `Dictionary` (no locking — user must ensure single-threaded access).

### Context Composition

`PipelineContext` composes five sub-contexts:

| Sub-Context | Properties |
|---|---|
| `RunIdentity` | `PipelineId`, `RunId`, `PipelineName`, `PipelineStartTimeUtc` |
| `ExecutionConfiguration` | Retry options, resilience policy, circuit breaker options, parallel execution flag |
| `Observability` | `LoggerFactory`, `Tracer`, `ObservabilityFactory`, `ExecutionObserver` |
| `NodeEnvironment` | Node execution scope registry, preconfigured nodes, DI ownership flag |
| `Lineage` | Lineage factory, lineage sink, pipeline lineage sink, lineage collector |

## PipelineRunner

### Creating a Runner

```csharp
// Simplest — all defaults
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
// No parameters, no cancellation
await runner.RunAsync<MyPipeline>();

// With parameters
var context = new PipelineContext(new PipelineContextConfiguration(
    Parameters: new Dictionary<string, object> { ["inputPath"] = "/data/file.csv" }
));
await runner.RunAsync<MyPipeline>(context);

// With a definition instance (constructor-injected parameters)
var definition = new MyPipeline("/data/file.csv");
await runner.RunAsync(definition, context, cts.Token);

// From IServiceProvider
await serviceProvider.RunPipelineAsync<MyPipeline>();
```

## Optimization Profiles

| Profile | Dictionary Type | Auto-Configured Retry | Analyzer Rules | Use Case |
|---|---|---|---|---|
| `Default` | `ConcurrentDictionary` | 3 retries (exp. backoff + jitter) | Suppressed NP9103-9107 | Prototyping, low-medium throughput |
| `HighThroughput` | Pooled `Dictionary` | None | All active | Millions of items/second |

```csharp
builder.WithOptimizationProfile(PipelineOptimizationProfile.HighThroughput);
```

## Pipeline Execution Lifecycle

1. **Build** — `PipelineBuilder.Build()` validates the graph, freezes collections, computes SHA256 hash
2. **Setup** — `RuntimePipelineBinder` resolves stream contracts, `NodeInstantiationService` creates node instances, `NodeRegistrationPlanner` compiles execution plans
3. **Execution** — `PipelineExecutionOrchestrator` walks graph in topological order, `NodeExecutor` processes data through each node
4. **Cleanup** — `PipelineContext.DisposeAsync()` disposes all registered async disposables, returns pooled dictionaries
