# Resilience API Reference

## ResilienceDecision Enum

Every failure is resolved to one of these decisions. Each policy method accepts only some of them.

| Value | Accepted by | Meaning |
|---|---|---|
| `Fail` | All layers | Stop execution, surface the failure |
| `Retry` | L1 item, L3 node | Retry the current item / execute the node again |
| `Skip` | L1 item | Skip the current item, continue with the next |
| `DeadLetter` | L1 item | Route the item to the dead-letter sink, continue |
| `RestartNode` | L2 stream | Restart the failed node's output stream from its checkpoint |
| `ContinueWithoutNode` | L2 stream | End the node's stream and let the rest of the pipeline continue |

## IResiliencePolicy

The central decision point, in the `NPipeline.Reliability` namespace. It has one method per layer:

```csharp
public interface IResiliencePolicy
{
    // L1: an item's transform failed.
    ValueTask<ResilienceDecision> DecideItemFailureAsync<TIn>(
        ItemFailure<TIn> failure, CancellationToken cancellationToken);

    // L2: a transform node's output stream failed.
    ValueTask<ResilienceDecision> DecideRestartAsync(
        StreamFailure failure, CancellationToken cancellationToken);

    // L3: a node's execution failed.
    ValueTask<ResilienceDecision> DecideNodeFailureAsync(
        NodeFailure failure, CancellationToken cancellationToken);
}
```

## Failure Structs

Each method receives a read-only record struct describing the failure. All of them carry the `PipelineContext` as `Context`.

### ItemFailure<TIn> (L1)

| Property | Description |
|---|---|
| `Item`, `Node`, `NodeId`, `Exception` | The item, the node, and what went wrong |
| `Attempt` | The 1-based number of the attempt that failed |
| `MaxRetries` | The node's `ItemRetry.MaxRetries` |
| `IsTransient` | Whether the node's `ItemRetry.Classifier` judged the failure transient |
| `IsBreakerOpen` | Whether an open circuit breaker refused the attempt (instead of the attempt being made and failing) |
| `CanRetry` | `true` when the failure is transient, the breaker didn't refuse the attempt, and retries remain |

### StreamFailure (L2)

| Property | Description |
|---|---|
| `NodeId`, `Exception` | The node and what went wrong |
| `Attempt`, `MaxRestarts` | The 1-based number of the run that failed, and the node's `NodeRestart.MaxRestarts` |
| `Checkpoint` | The input index a restart resumes from |
| `Delivered` | The outputs the node has delivered so far, across all runs |
| `CanRestart` | `true` when restarts remain |

### NodeFailure (L3)

| Property | Description |
|---|---|
| `Definition`, `Node`, `NodeId`, `Exception` | The node and what went wrong |
| `Attempt`, `MaxRetries`, `IsTransient` | From the node's `NodeRetry` options |
| `InputConsumed` | Whether the node read input before it failed. A node that has read input isn't executed again, whatever the policy answers |
| `CanRetry` | `true` when the failure is transient, retries remain, and the node hasn't read input |

## DefaultResiliencePolicy

The policy used when none is registered. It follows the node's options and adds no rules of its own:

- **Item failure:** `Retry` while `CanRetry`; otherwise the node's `OnItemFailure` (`Fail`, `Skip`, or `DeadLetter`). An attempt refused by an open breaker fails, even when `OnItemFailure` is `Skip` or `DeadLetter`.
- **Stream failure:** `RestartNode` while `CanRestart`, otherwise `Fail`.
- **Node failure:** `Retry` while `CanRetry`, otherwise `Fail`.

## ResiliencePolicyBase

Derive from this to override only the decisions you care about; the others follow `DefaultResiliencePolicy`:

```csharp
using NPipeline.Reliability;

public sealed class PatientWithRateLimits : ResiliencePolicyBase
{
    public override ValueTask<ResilienceDecision> DecideItemFailureAsync<TIn>(
        ItemFailure<TIn> failure, CancellationToken cancellationToken)
    {
        if (failure.Exception is HttpRequestException { StatusCode: HttpStatusCode.TooManyRequests } && failure.Attempt <= 10)
            return ValueTask.FromResult(ResilienceDecision.Retry);

        return base.DecideItemFailureAsync(failure, cancellationToken);
    }
}
```

The node's options reach the policy as advice (`MaxRetries`, `CanRetry`); the runtime never overrides the answer. A policy that answers `Retry` or `RestartNode` more than 100 times for one unit of work fails the node with an `InvalidOperationException` rather than looping forever.

## PipelineResilienceOptions

```csharp
public sealed record PipelineResilienceOptions
{
    public ItemRetryOptions ItemRetry { get; init; } = ItemRetryOptions.None;
    public NodeRestartOptions NodeRestart { get; init; } = NodeRestartOptions.None;
    public NodeRetryOptions NodeRetry { get; init; } = NodeRetryOptions.None;
    public CircuitBreakerOptions? CircuitBreaker { get; init; }
    public ItemFailureAction OnItemFailure { get; init; } = ItemFailureAction.Fail;
    public TimeProvider Time { get; init; } = TimeProvider.System;
}
```

### Configuration Methods

```csharp
// Pipeline-wide options
builder.WithResilience(options => options with
{
    ItemRetry = ItemRetryOptions.Default with { MaxRetries = 5 },
    OnItemFailure = ItemFailureAction.DeadLetter,
});

// Per-node options, derived from the pipeline's
builder.WithResilience(enrich, options => options with
{
    NodeRestart = new NodeRestartOptions { MaxRestarts = 2 },
});
```

> [!NOTE]
> Setting `ItemRetry`, `NodeRestart`, or `CircuitBreaker` on a source, sink, aggregate, or join node is a build error, because none of them can apply there. The NP9204 analyzer reports the same mistake at compile time.

### Profile-Aware Defaults

`PipelineResilienceOptions.ForProfile(profile)` returns profile-appropriate defaults:

| Option | Default Profile | HighThroughput Profile |
|---|---|---|
| `ItemRetry` | `ItemRetryOptions.Default` (3 retries, exponential backoff) | `ItemRetryOptions.None` |
| `NodeRestart` | None | None |
| `NodeRetry` | None | None |
| `CircuitBreaker` | `null` | `null` |

## ItemRetryOptions, NodeRestartOptions, NodeRetryOptions

```csharp
public sealed record ItemRetryOptions
{
    public static ItemRetryOptions None { get; } = new() { MaxRetries = 0 };
    public static ItemRetryOptions Default { get; } = new()
    {
        MaxRetries = 3,
        Backoff = RetryBackoff.Exponential(TimeSpan.FromMilliseconds(200), maxDelay: TimeSpan.FromSeconds(30)),
    };
    public int MaxRetries { get; init; }
    public RetryBackoff Backoff { get; init; } = RetryBackoff.None;
    public RetryClassifier Classifier { get; init; } = RetryClassifier.Default;
}

public sealed record NodeRestartOptions
{
    public static NodeRestartOptions None { get; } = new() { MaxRestarts = 0 };
    public int MaxRestarts { get; init; }
    public RetryBackoff Backoff { get; init; } =
        RetryBackoff.Exponential(TimeSpan.FromSeconds(1), maxDelay: TimeSpan.FromSeconds(30));
    public int MaxReplayWindow { get; init; } = 10_000;   // Backpressure bound, never an error
    public int? ResetAfterItems { get; init; }
}

public sealed record NodeRetryOptions
{
    public static NodeRetryOptions None { get; } = new() { MaxRetries = 0 };
    public int MaxRetries { get; init; }
    public RetryBackoff Backoff { get; init; } =
        RetryBackoff.Exponential(TimeSpan.FromSeconds(1), maxDelay: TimeSpan.FromSeconds(30));
    public RetryClassifier Classifier { get; init; } = RetryClassifier.Default;
}
```

## RetryBackoff and RetryClassifier

`RetryBackoff` is a record struct with a factory method per curve. The retry number is 1-based.

| Factory | Delay before retry *n* |
|---|---|
| `RetryBackoff.None` | Zero |
| `RetryBackoff.Constant(delay)` | `delay` |
| `RetryBackoff.Linear(step, maxDelay)` | `step` × *n* |
| `RetryBackoff.Exponential(baseDelay, factor, maxDelay)` | `baseDelay` × `factor`^(*n* - 1) |
| `RetryBackoff.Custom(delayForRetry)` | `delayForRetry(n)` |

`RetryClassifier.Default` treats timeouts, I/O, socket and transient database errors, and HTTP 408, 429 and 5xx responses as transient; `RetryClassifier.All` retries everything except the pipeline's own cancellation. Add rules with `Transient<TException>()` and `Permanent<TException>()`.

## Dead-Letter Types

```csharp
public interface IDeadLetterSink
{
    Task HandleAsync(
        DeadLetterEnvelope envelope,
        PipelineContext context,
        CancellationToken cancellationToken);
}

public sealed record DeadLetterEnvelope(
    object Item,
    Exception Error,
    NodeFailureAttribution Attribution);

public sealed record NodeFailureAttribution(
    string OriginNodeId,
    string DecisionNodeId,
    Guid OriginPipelineId,
    Guid DecisionPipelineId,
    Guid? RunId = null,
    Guid? CorrelationId = null,
    int RetryCount = 0);
```

Register a sink via `builder.AddDeadLetterSink<MyDeadLetterSink>()` or `builder.AddDeadLetterSink(sinkInstance)`. Built-in: `BoundedInMemoryDeadLetterSink` (default capacity 1,000; throws when full to prevent unbounded growth).

> [!WARNING]
> A policy that returns `DeadLetter` needs a dead-letter sink. If none is configured, the node fails with a `DeadLetterSinkNotConfiguredException` (NP0424). The item is never dropped silently.

## Node Restart and the Replay Window

Node restart resumes at the node's checkpoint, the first input item whose outcome has not been delivered, so already-delivered items aren't processed or delivered again. `NodeRestartOptions.MaxReplayWindow` bounds how many items between the checkpoint and the last-read item are held: when that many are held, the node stops reading input until the checkpoint advances. It is a backpressure bound, never an error, and does not limit the length of the input.

## Fluent Policy Builder

`ResiliencePolicyBuilder` (namespace `NPipeline.ErrorHandling`) builds an item-decision policy without a class:

```csharp
using NPipeline.ErrorHandling;
using NPipeline.Reliability;

var policy = ResiliencePolicyBuilder
    .ForNode<ParseOrderTransform, string>()
    .On<TimeoutException>().Retry(maxRetries: 3)
    .On<FormatException>().Skip()
    .OnAny().Fail()
    .Build();

builder.AddResiliencePolicy(policy);
```

Rules are checked in order; the first match wins. `OnAny()` must be last, or `Build()` throws `InvalidOperationException`. The builder's policies make item decisions only; restart and node decisions follow the node's options. A rule such as `.OnAny().Retry(3)` also retries permanent failures because the builder doesn't consult the classifier.

Shortcuts: `ResiliencePolicyBuilder.RetryAlways<TNode,TData>(maxRetries)`, `RetryOn<TNode,TData,TException>(maxRetries)`, `SkipAlways<TNode,TData>()`, `DeadLetterAlways<TNode,TData>()`.
