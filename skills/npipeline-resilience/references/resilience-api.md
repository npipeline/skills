# Resilience API Reference

## ResilienceDecision Enum

Every failure is resolved to one of these decisions:

| Value | Scope | Meaning |
|---|---|---|
| `Fail` | Item/Node/Pipeline | Stop execution, surface the failure |
| `Retry` | Item | Retry the current item |
| `Skip` | Item | Skip the current item, continue with next |
| `DeadLetter` | Item | Route item to dead-letter sink, continue |
| `RestartNode` | Node | Restart the failed node (replays materialized items) |
| `ContinueWithoutNode` | Node | Continue pipeline without the failed node |

## IResiliencePolicy

The central decision point for all failures:

```csharp
public interface IResiliencePolicy
{
    // Node-level failure (entire node failed)
    Task<ResilienceDecision> DecideNodeFailureAsync(
        NodeDefinition nodeDefinition,
        INode node,
        Exception exception,
        PipelineContext context,
        CancellationToken cancellationToken);

    // Pipeline-level stream failure
    Task<ResilienceDecision> DecidePipelineFailureAsync(
        string nodeId,
        Exception exception,
        PipelineContext context,
        CancellationToken cancellationToken);

    // Item-level failure (single item failed in transform)
    Task<ResilienceDecision> DecideItemFailureAsync<TIn, TOut>(
        ITransformNode<TIn, TOut> node,
        TIn failedItem,
        Exception exception,
        PipelineContext context,
        string nodeId,
        int retryAttempt,
        CancellationToken cancellationToken);

    // Retry delay calculation
    ValueTask<TimeSpan> GetRetryDelayAsync(
        PipelineContext context,
        int attemptNumber,
        CancellationToken cancellationToken);

    // Circuit breaker for a specific node
    IResilienceCircuitBreaker? GetCircuitBreaker(
        PipelineContext context, string nodeId);
}
```

## ResiliencePolicyBase

Abstract base class that simplifies policy implementation. Only override the methods you need — the base returns `Fail` for all decisions by default.

```csharp
public class MyPolicy : ResiliencePolicyBase
{
    public override async Task<ResilienceDecision> DecideItemFailureAsync<TIn, TOut>(
        ITransformNode<TIn, TOut> node, TIn failedItem, Exception exception,
        PipelineContext context, string nodeId, int retryAttempt, CancellationToken ct)
    {
        return exception switch
        {
            HttpRequestException when retryAttempt < 3 => ResilienceDecision.Retry,
            TimeoutException => ResilienceDecision.Retry,
            ValidationException => ResilienceDecision.Skip,
            _ => ResilienceDecision.DeadLetter
        };
    }

    protected override ValueTask<TimeSpan> GetRetryDelayAsync(
        PipelineContext ctx, int attempt, CancellationToken ct)
    {
        // Exponential backoff: 1s, 2s, 4s, 8s...
        return ValueTask.FromResult(TimeSpan.FromSeconds(Math.Pow(2, attempt - 1)));
    }
}
```

## PipelineRetryOptions

```csharp
public sealed record PipelineRetryOptions(
    int MaxItemRetries = 0,                    // 0 = no retry
    int? MaxMaterializedItems = null,          // null = unbounded
    RetryDelayStrategyConfiguration? DelayStrategyConfiguration = null,
    int MaxNodeRestartAttempts = 3,
    int MaxSequentialNodeAttempts = 5)
```

### Configuration Methods

```csharp
// Simple item retries
builder.WithRetryOptions(o => o with { MaxItemRetries = 3 });

// With backoff
builder.WithRetryOptions(o => o with
{
    MaxItemRetries = 5,
    DelayStrategyConfiguration = RetryDelayStrategyConfigurationExtensions.DefaultExponentialBackoffWithJitter
});

// Per-node retry
handle.WithRetries(builder, 3);

// Node restart with materialization cap
builder.WithRetryOptions(o => o with
{
    MaxNodeRestartAttempts = 2,
    MaxMaterializedItems = 100_000
});
```

### Profile-Aware Defaults

`PipelineRetryOptions.ForProfile(profile)` returns profile-appropriate defaults:

| Option | Default Profile | HighThroughput Profile |
|---|---|---|
| `MaxItemRetries` | 3 | 0 |
| `MaxMaterializedItems` | 10,000 | null |
| `DelayStrategy` | Exponential + Jitter | null |
| `MaxNodeRestartAttempts` | 3 | 3 |
| `MaxSequentialNodeAttempts` | 5 | 5 |

## DeadLetterEnvelope

Items that result in `DeadLetter` decisions are wrapped:

```csharp
public sealed record DeadLetterEnvelope(
    object Item,
    Exception Error,
    NodeFailureAttribution Attribution)
```

`NodeFailureAttribution` records the origin and decision details:

```csharp
public sealed record NodeFailureAttribution(
    string OriginNodeId,
    string DecisionNodeId,
    Guid OriginPipelineId,
    Guid DecisionPipelineId,
    Guid? RunId = null,
    Guid? CorrelationId = null,
    int RetryCount = 0)
```

## IDeadLetterSink

```csharp
public interface IDeadLetterSink
{
    Task HandleAsync(
        DeadLetterEnvelope envelope,
        PipelineContext context,
        CancellationToken cancellationToken);
}
```

Register via:

```csharp
builder.AddDeadLetterSink<MyDeadLetterSink>();
```

Built-in: `BoundedInMemoryDeadLetterSink` — in-memory dead letter with configurable capacity cap.

## Materialization

Node restart requires replaying past items. `ResilientExecutionStrategy` materializes items for replay:

- Forward-only streams are detected and materialized into `CappedReplayableDataStream<T>`
- `MaxMaterializedItems` caps the materialization buffer
- Overflow policy for exceeding capacity: oldest items dropped

Without materialization, forward-only streams cannot be replayed, so `RestartNode` decisions on such streams will be denied.
