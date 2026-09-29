---
name: npipeline-resilience
description: Use when the user wants to configure NPipeline error handling, retries, circuit breakers, dead-letter queues, or resilience policies. Covers PipelineResilienceOptions (ItemRetry, NodeRestart, NodeRetry, CircuitBreaker), IResiliencePolicy at the three layers, ResilienceDecision, RetryBackoff and RetryClassifier, IDeadLetterSink, and the Default/HighThroughput profile retry defaults. Use when user mentions "retry", "error handling", "circuit breaker", "dead letter", "dead-letter queue", "resilience policy", or "node restart".
npipelineVersion: "0.67.0"
lastVerified: "2026-09-29"
---

# NPipeline Resilience and Error Handling

This skill covers NPipeline's resilience subsystem: retries, circuit breakers, resilience policies, dead-letter queues, and node restart.

## Workflow

When the user wants to handle errors, determine the failure scope (item, stream, or node) and choose the appropriate mechanism.

### Phase 1: Understand the Failure Model

NPipeline has a three-layer resilience model:

| Layer | Option | Applies to | What it repeats | Off by default? |
|---|---|---|---|---|
| **L1: item retry** | `ItemRetry` | Transform nodes | One item's `TransformAsync` | No. The `Default` profile retries transient failures three times. |
| **L2: node restart** | `NodeRestart` | Transform nodes | The node's output stream, resumed from its checkpoint | Yes |
| **L3: node retry** | `NodeRetry` | Any node | The node's setup, before it reads input | Yes |

All three live on `PipelineResilienceOptions` (`NPipeline.Reliability`). Every decision flows through `IResiliencePolicy`; with no policy registered, `DefaultResiliencePolicy` follows the node's options exactly.

> [!IMPORTANT]
> Resilience is configured with `builder.WithResilience(o => o with { ... })` over `PipelineResilienceOptions`. `WithRetryOptions`, `PipelineRetryOptions`, and `MaxMaterializedItems` are not part of this API.

### Phase 2: Configure Retries

```csharp
using NPipeline.Reliability;

builder.WithResilience(options => options with
{
    ItemRetry = options.ItemRetry with
    {
        MaxRetries = 5,
        Backoff = RetryBackoff.Exponential(TimeSpan.FromMilliseconds(500), maxDelay: TimeSpan.FromSeconds(30)),
    },
});
```

Under the `Default` optimization profile, transient item failures are already retried three times (`ItemRetryOptions.Default`). `HighThroughput` retries nothing. Per-node:

```csharp
builder.WithResilience(enrich, options => options with
{
    ItemRetry = ItemRetryOptions.Default with { MaxRetries = 10 },
});
```

### Phase 3: Add a Circuit Breaker (optional)

```csharp
using NPipeline.Reliability;

builder.WithResilience(transform, options => options with
{
    CircuitBreaker = new CircuitBreakerOptions
    {
        ConsecutiveFailures = 5,
        OpenDuration = TimeSpan.FromSeconds(30),
    },
});
```

### Phase 4: Write a Custom Resilience Policy (for advanced cases)

When the decision depends on the exception type or item content, derive from `ResiliencePolicyBase` and override only the decisions you care about:

```csharp
using NPipeline.Reliability;

public sealed class DeadLetterInvalidOrders : ResiliencePolicyBase
{
    public override ValueTask<ResilienceDecision> DecideItemFailureAsync<TIn>(
        ItemFailure<TIn> failure, CancellationToken cancellationToken)
    {
        return failure.Exception is ValidationException
            ? ValueTask.FromResult(ResilienceDecision.DeadLetter)
            : base.DecideItemFailureAsync(failure, cancellationToken);
    }
}

builder.AddResiliencePolicy(new DeadLetterInvalidOrders());
```

Consult `references/resilience-api.md` for the complete `IResiliencePolicy` interface, the failure structs, and `ResilienceDecision`.
Consult `references/circuit-breaker.md` for circuit breaker configuration details.
