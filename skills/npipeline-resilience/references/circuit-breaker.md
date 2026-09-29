# Circuit Breaker Reference

A circuit breaker watches the item attempts of a transform node. When enough attempts fail transiently, the breaker opens, and the node stops making attempts for a while.

## How the Breaker Works

```
Closed --(trip condition met)--> Open
Open --(OpenDuration passes)--> Half-Open
Half-Open --(ProbeSuccesses probes succeed)--> Closed
Half-Open --(a probe fails transiently)--> Open
```

| State | Behavior |
|---|---|
| **Closed** | Every attempt is made. Transient failures count toward the trip conditions. |
| **Open** | No attempt is made. What happens to the item depends on `WhenOpen`. |
| **Half-Open** | Up to `HalfOpenProbes` attempts (probes) are made at a time to test recovery. |

Only failures the node's `ItemRetry.Classifier` judges transient count against the breaker. A permanent failure neither trips the breaker nor resets its count. Cancellation doesn't count either.

## CircuitBreakerOptions

```csharp
public sealed record CircuitBreakerOptions
{
    public static CircuitBreakerOptions Default { get; } = new();

    public int? ConsecutiveFailures { get; init; } = 5;      // null disables this condition
    public double? FailureRate { get; init; }                // > 0 and <= 1; null disables
    public int MinimumCalls { get; init; } = 20;
    public TimeSpan Window { get; init; } = TimeSpan.FromSeconds(30);
    public TimeSpan OpenDuration { get; init; } = TimeSpan.FromSeconds(30);
    public int HalfOpenProbes { get; init; } = 1;
    public int ProbeSuccesses { get; init; } = 1;
    public BreakerOpenBehavior WhenOpen { get; init; } = BreakerOpenBehavior.Fail;
    public TimeSpan MaxPause { get; init; } = TimeSpan.FromMinutes(5);
}
```

The breaker trips when **any** configured condition is met. At least one of `ConsecutiveFailures` and `FailureRate` must be set, or `Validate()` throws. `Window`, `OpenDuration` and `MaxPause` must each be positive and at most `int.MaxValue` milliseconds (about 24.8 days). Setting `CircuitBreaker = null` (the default) turns the breaker off.

> [!NOTE]
> A breaker only guards transform nodes. Setting one on a source, sink, aggregate, or join node is a build error.

## Configuration

### Fast-Fail on Consecutive Failures

```csharp
using NPipeline.Reliability;

builder.WithResilience(transform, options => options with
{
    ItemRetry = ItemRetryOptions.Default,
    CircuitBreaker = new CircuitBreakerOptions
    {
        ConsecutiveFailures = 5,
        OpenDuration = TimeSpan.FromSeconds(30),
    },
});
```

After five transient failures in a row the breaker opens, stays open for 30 seconds, then lets one probe through. A successful probe closes it; a transient probe failure reopens it.

### Trip on a Failure Rate

```csharp
CircuitBreaker = new CircuitBreakerOptions
{
    ConsecutiveFailures = null,          // Rate only
    FailureRate = 0.5,                   // Half the attempts failed transiently
    MinimumCalls = 50,                   // Out of at least 50 attempts
    Window = TimeSpan.FromMinutes(1),    // Within the last minute
},
```

`MinimumCalls` prevents the breaker from tripping on a handful of early failures.

## What Happens While the Breaker Is Open

### Fail (default)

With `WhenOpen = BreakerOpenBehavior.Fail`, an attempt the open breaker refuses isn't made. The item fails with a `CircuitBreakerOpenException` whose `NodeId` and `State` properties say which breaker refused it. The policy sees `ItemFailure.IsBreakerOpen = true`, and `DefaultResiliencePolicy` fails the node even when `OnItemFailure` is `Skip` or `DeadLetter`, so one outage doesn't drain the input.

### Pause (opt-in)

```csharp
CircuitBreaker = new CircuitBreakerOptions
{
    ConsecutiveFailures = 5,
    OpenDuration = TimeSpan.FromSeconds(30),
    WhenOpen = BreakerOpenBehavior.Pause,   // Wait for the breaker instead of failing
    MaxPause = TimeSpan.FromMinutes(10),    // But never longer than 10 minutes for one attempt
},
```

With `Pause`, a refused attempt waits until the breaker lets a probe through and then makes the attempt as a probe. While it waits, the node reads no more input, so upstream nodes are held back by backpressure. The pause is bounded by `MaxPause` and by pipeline cancellation.

## Breaker Lifetime

A node's breaker lives as long as the `PipelineFactory` that built the pipeline, not just one run. `AddNPipeline()` registers the factory as a singleton, and `PipelineRunner.Create()` creates one per runner, so reusing a runner reuses its breakers. Each pipeline definition type has its own breakers, one per node.

## Testing with a Fake Clock

The breaker reads time from `PipelineResilienceOptions.Time`. Set it to a `FakeTimeProvider` from `Microsoft.Extensions.TimeProvider.Testing` and advance the clock instead of waiting out `OpenDuration`:

```csharp
var time = new FakeTimeProvider();

builder.WithResilience(transform, options => options with
{
    CircuitBreaker = new CircuitBreakerOptions { ConsecutiveFailures = 2, OpenDuration = TimeSpan.FromMinutes(1) },
    Time = time,
});

// ... run until the breaker opens ...

time.Advance(TimeSpan.FromMinutes(1)); // The next attempt is a probe.
```
