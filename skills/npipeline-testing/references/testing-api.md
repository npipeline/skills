# Testing API Reference

## Packages

```
dotnet add package NPipeline.Extensions.Testing
dotnet add package NPipeline.Extensions.Testing.FluentAssertions
dotnet add package NPipeline.Extensions.Testing.AwesomeAssertions
```

## PipelineTestHarness<TPipeline>

Fluent builder for integration testing:

```csharp
var result = await new PipelineTestHarness<MyPipeline>()
    .WithParameter("key", value)           // Add to Parameters dictionary
    .WithParameters(dict)                  // Multiple parameters at once
    .WithContextItem("key", value)         // Add to Items dictionary
    .WithExecutionObserver(myObserver)     // Custom execution observer
    .CaptureErrors()                       // Don't throw on failure — capture errors
    .RunAsync(cancellationToken);          // Execute and return result

// Result
var result = await harness.RunAsync();
```

## PipelineExecutionResult

```csharp
public sealed record PipelineExecutionResult
{
    bool Success
    TimeSpan Duration
    IReadOnlyList<Exception> Errors
    PipelineContext Context
}
```

### Assertion Methods (FluentAssertions / AwesomeAssertions)

```csharp
// Success/failure assertions
result.AssertSuccess();                    // Pipeline completed successfully
result.AssertFailure();                    // Pipeline failed

// Error assertions
result.AssertNoErrors();                   // No errors recorded
result.AssertErrorOfType<T>();            // At least one error of type T
result.AssertErrorCount(3);               // Exact error count

// Timing assertions
result.AssertCompletedWithin(TimeSpan.FromSeconds(5)); // Duration check

// Data extraction
var sink = result.GetSink<T>();           // Get first InMemorySink<T> from context
var items = sink.Items;                   // All items that arrived at the sink

// Context inspection
context.TryGetContextItem<T>("key", out val);
```

## Testing Nodes

### InMemorySourceNode<T>

Parameterless, context-backed source for test data. Extension methods on `PipelineBuilder` (from `NPipeline.Extensions.Testing`):

```csharp
// Via builder extension
builder.AddInMemorySource<Order>(orders);   // IEnumerable<T> - items provided directly
builder.AddInMemorySource<Order>();         // Uses context to resolve source data
builder.AddInMemorySourceWithDataFromContext<Order>(context, orders); // Set context data
```

### InMemorySinkNode<T>

Collector sink — items are captured internally and accessible via a snapshot:

```csharp
var sink = new InMemorySinkNode<Order>();
builder.AddInMemorySink<Order>(sink);

// After pipeline runs
var collected = sink.Items; // IReadOnlyList<Order> (snapshot)
```

### Additional Testing Nodes

| Node | Purpose |
|---|---|
| `MockNode<TIn, TOut>` | Delegate-based transform for mocking |
| `PassThroughTransformNode<TIn, TOut>` | Type casting (no logic) |
| `ExceptionThrowingNode<TIn>` | Always throws for error path testing |

## Error Capture

Use `CaptureErrors()` on the test harness to capture exceptions instead of throwing them. The harness internally wraps the resilience policy with error-capturing logic:

```csharp
var result = await new PipelineTestHarness<MyPipeline>()
    .CaptureErrors(ResilienceDecision.Skip)  // Skip failed items, capture exceptions
    .RunAsync();

// All captured exceptions are available
result.Errors.Should().Contain(e => e is ValidationException);

// You can also specify the decision per failure type
var result = await new PipelineTestHarness<MyPipeline>()
    .CaptureErrors(ResilienceDecision.Retry)  // Retry failures, but still capture
    .RunAsync();
```

The error-capturing mechanism is internal to the test harness — you interact with it via `CaptureErrors()`, not by creating a `CapturingResiliencePolicy` directly.

## TestPipelineRunner

Simplified runner that returns results directly:

```csharp
var runner = new TestPipelineRunner();
var (success, result) = await runner.RunAndGetResultAsync<MyPipeline, EnrichedOrder>();
// result is the collected sink data
```

## Testing Best Practices

1. **Unit test nodes directly** — Call `TransformAsync` with controlled inputs.
2. **Integration test pipelines** — Use `PipelineTestHarness` with `InMemorySource`/`InMemorySink`.
3. **Test error paths** — Use `CaptureErrors()` on the harness to capture exceptions during execution.
4. **Use `CaptureErrors()`** — Prevents test from crashing on expected failures.
5. **Parameterize tests** — Use `[Theory]` with `[InlineData]` for data-driven node testing.
6. **Mock dependencies** — Use FakeItEasy or Moq for external service dependencies in nodes.
7. **Name tests clearly** — Convention: `MethodName_Condition_ExpectedBehavior`.
