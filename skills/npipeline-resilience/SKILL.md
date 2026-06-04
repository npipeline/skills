---
name: npipeline-resilience
description: Use when the user wants to configure NPipeline error handling, retry strategies, circuit breakers, dead-letter queues, or resilience policies. Covers PipelineRetryOptions, IResiliencePolicy, ResilienceDecision enum, circuit breaker configuration, IDeadLetterSink, node restart, materialization, and the Default/HighThroughput profile retry defaults. Use when user mentions "retry", "error handling", "circuit breaker", "dead letter", "dead-letter queue", "resilience policy", or "node restart".
npipelineVersion: "0.52.0"
lastVerified: "2026-06-04"
---

# NPipeline Resilience and Error Handling

This skill covers NPipeline's resilience subsystem: retries, circuit breakers, resilience policies, dead-letter queues, and node restart.

## Workflow

When the user wants to handle errors, determine the failure scope (item, node, or pipeline) and choose the appropriate mechanism.

### Phase 1: Understand the Failure Model

NPipeline has a three-tier failure model:

| Level | Scope | Available Decisions |
|---|---|---|
| **Item** | One item fails in a transform | `Retry`, `Skip`, `DeadLetter` |
| **Node** | Entire node fails (stream error) | `RestartNode`, `ContinueWithoutNode`, `Fail` |
| **Pipeline** | Critical failure or circuit breaker trips | `Fail` (pipeline stops) |

Every resilience decision flows through `IResiliencePolicy`. The default policy returns `Fail` for everything — users must opt into recovery.

### Phase 2: Configure Retries

The simplest path. Configure per-item retries with optional backoff:

```csharp
builder.WithRetryOptions(o => o with { MaxItemRetries = 3 });
```

In `Default` optimization profile, these are auto-configured: 3 retries, exponential backoff + full jitter, 10,000-item materialization cap.

### Phase 3: Add Circuit Breakers (optional)

Prevent cascading failures:

```csharp
builder.WithCircuitBreaker(failureThreshold: 5, openDuration: TimeSpan.FromMinutes(1));
```

### Phase 4: Write a Custom Resilience Policy (for advanced cases)

When you need custom decision logic (e.g., different treatment for different exception types):

```csharp
public class MyPolicy : ResiliencePolicyBase
{
    public override async Task<ResilienceDecision> DecideItemFailureAsync<TIn, TOut>(...)
    {
        if (exception is TransientException)
            return ResilienceDecision.Retry;
        if (exception is ValidationException)
            return ResilienceDecision.Skip;
        return ResilienceDecision.DeadLetter;
    }
}

builder.AddResiliencePolicy<MyPolicy>();
```

Consult `references/resilience-api.md` for the complete `IResiliencePolicy` interface and `ResilienceDecision` enum.
Consult `references/circuit-breaker.md` for circuit breaker configuration details.
