# Circuit Breaker Reference

## PipelineCircuitBreakerOptions

```csharp
public sealed record PipelineCircuitBreakerOptions(
    int FailureThreshold,                              // Default: 5
    TimeSpan OpenDuration,                             // Default: 1 minute
    TimeSpan SamplingWindow,                           // Default: 5 minutes
    bool Enabled = true,
    CircuitBreakerThresholdType ThresholdType = ConsecutiveFailures,
    double FailureRateThreshold = 0.5,
    int HalfOpenSuccessThreshold = 1,
    int HalfOpenMaxAttempts = 5,
    bool TrackOperationsInWindow = true)
```

## Threshold Types

| Type | Behavior | When to Use |
|---|---|---|
| `ConsecutiveFailures` | Trips after N consecutive failures | Simple, fast-to-trip |
| `RollingWindowCount` | Trips after N failures in sampling window | Bursty failures |
| `RollingWindowRate` | Trips when failure rate exceeds `FailureRateThreshold` | Rate-based protection |
| `Hybrid` | Both count AND rate must exceed threshold | Conservative protection |

## Circuit Breaker States

The circuit breaker follows the standard state machine:

```
Closed ----(failures >= threshold)----> Open
  ^                                        |
  |                                        v
  +---(successes >= threshold)-- Half-Open <----(open duration elapsed)
```

- **Closed** — Normal operation. Failures are counted.
- **Open** — All operations fail fast. Duration: `OpenDuration`.
- **Half-Open** — Limited operations (`HalfOpenMaxAttempts`) are attempted. If `HalfOpenSuccessThreshold` successes occur, transitions to Closed. Otherwise, re-opens.

## Configuration Examples

### Fast-Fail on Consecutive Failures

```csharp
builder.WithCircuitBreaker(
    failureThreshold: 3,
    openDuration: TimeSpan.FromSeconds(30));
```

### Rolling Window Rate-Based

```csharp
builder.WithCircuitBreaker(
    failureThreshold: 10,
    openDuration: TimeSpan.FromMinutes(2),
    thresholdType: CircuitBreakerThresholdType.RollingWindowRate,
    failureRateThreshold: 0.3,
    samplingWindow: TimeSpan.FromMinutes(5));
```

### Disable Circuit Breaker

```csharp
// Circuit breaker is not enabled by default.
// To explicitly disable: use PipelineCircuitBreakerOptions.Disabled
builder.WithCircuitBreaker(PipelineCircuitBreakerOptions.Disabled);
```

## Memory Management

`CircuitBreakerMemoryManagementOptions` controls cleanup:

```csharp
public sealed record CircuitBreakerMemoryManagementOptions(
    TimeSpan? CleanupInterval,
    TimeSpan? InactivityThreshold,
    int MaxTrackedBreakers)
```

Unused circuit breakers that haven't been accessed within `InactivityThreshold` are cleaned up automatically.

## Fluent Error Handler

`ResiliencePolicyBuilder` provides factory methods for creating targeted resilience policies using a fluent builder pattern:

```csharp
// Node-scoped policy for item-level failures
var policy = ResiliencePolicyBuilder
    .ForNode<MyTransform, Order>()
    .On<HttpRequestException>().Retry()
    .On<TimeoutException>().Retry()
    .On<ValidationException>().Skip()
    .On<InvalidOperationException>().DeadLetter()
    .Build();

builder.AddResiliencePolicy(policy);
```

First-match wins — exceptions are checked in the order they are registered.

For simple retry-always behavior:

```csharp
builder.AddResiliencePolicy(
    ResiliencePolicyBuilder.RetryAlways<MyTransform, Order>(maxRetries: 3));
```
