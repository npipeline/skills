# Testing API Reference

## Packages

```
dotnet add package NPipeline.Extensions.Testing
dotnet add package NPipeline.Extensions.Testing.FluentAssertions
dotnet add package NPipeline.Extensions.Testing.AwesomeAssertions
```

## PipelineTestHarness<TPipeline>

Fluent builder for integration testing. `TPipeline` must be `IPipelineDefinition, new()`.

```csharp
var result = await new PipelineTestHarness<MyPipeline>()
    .WithParameter("key", value)           // Add to Parameters dictionary
    .WithParameters(dict)                  // Multiple parameters at once
    .WithContextItem("key", value)         // Add to Items dictionary
    .WithExecutionObserver(myObserver)     // Custom execution observer
    .CaptureErrors()                       // Don't throw on failure; capture errors
    .RunAsync(cancellationToken);          // Execute and return result
```

The harness owns a `PipelineContext` (accessible as `harness.Context`) and a runner (default `PipelineRunner.Create()`).

## PipelineExecutionResult

```csharp
public record PipelineExecutionResult(
    bool Success,
    TimeSpan Duration,
    IReadOnlyList<Exception> Errors,
    PipelineContext Context);
```

### Assertion Methods (both assertion packages)

```csharp
result.AssertSuccess();                    // Pipeline completed with no uncaught exception
result.AssertFailure();                    // Pipeline failed
result.AssertNoErrors();                   // No errors recorded
result.AssertErrorOfType<TException>();    // At least one error of type T
result.AssertErrorCount(3);                // Exact error count
result.AssertCompletedWithin(TimeSpan.FromSeconds(5));

var sink = result.GetSink<InMemorySinkNode<Order>>();  // First matching sink from context
var items = sink.Items;

result.TryGetContextItem<T>("key", out var value);
```

The assertion methods are extension methods (`PipelineExecutionResultExtensions`) and do not require either assertion package. The `InMemorySinkNode<T>` helpers (`ShouldHaveReceived`, `ShouldContain`, `ShouldOnlyContain`, and so on) are provided by the FluentAssertions and AwesomeAssertions packages.

## Testing Nodes

### InMemorySourceNode<T>

Context-backed or list-backed source for test data. Extension methods on `PipelineBuilder` (from `NPipeline.Extensions.Testing`):

```csharp
builder.AddInMemorySource<Order>(orders);            // Items provided directly
builder.AddInMemorySource<Order>("orders");          // Named; resolves data from the context
builder.AddInMemorySourceWithDataFromContext<Order>(context, orders);

// Set the data the parameterless/named source resolves
context.SetSourceData(orders);
```

### InMemorySinkNode<T>

Collector sink. It registers itself in the context during execution, so `context.GetSink<T>()` finds it after the run:

```csharp
// Add a sink; retrievable from the context after the run
builder.AddInMemorySink<Order>("results");

// Or register it up front against a specific context
builder.AddInMemorySink<Order>(context);

// After the run
var sink = context.GetSink<InMemorySinkNode<Order>>();
var collected = sink.Items;    // IReadOnlyList<Order> (snapshot)
```

`InMemorySinkNode<T>` also exposes `Completion`, a `Task<IReadOnlyList<T>>` that completes with the items when the sink finishes.

### Additional Testing Nodes

| Node | Purpose |
|---|---|
| `MockNode<TIn, TOut>` | Delegate-based transform; ctor takes `Func<TIn, PipelineContext, CancellationToken, Task<TOut>>` |
| `PassThroughTransformNode<TIn, TOut>` | Identity / type-casting transform |
| `ExceptionThrowingNode<TIn>` | Always throws, for error path testing |
| `CapturingLogger` | Captures log entries for assertions |

## Error Capture

`CaptureErrors()` makes the harness capture exceptions instead of letting them fail the run. It wraps whichever policy the run resolves (a node's, the pipeline's, or the context's), so that policy still runs first.

```csharp
var result = await new PipelineTestHarness<MyPipeline>()
    .CaptureErrors(ResilienceDecision.Skip)  // Apply Skip to captured item failures
    .RunAsync();

result.Errors.Should().Contain(e => e is ValidationException);
```

`CaptureErrors` accepts any `ResilienceDecision` (default `Skip`); the value becomes the decision the wrapper returns for captured failures. You interact with it through `CaptureErrors()`, not by creating a capturing policy directly.

## TestPipelineRunner

A helper that runs a pipeline and returns the items collected by an `InMemorySinkNode<T>`. It requires an `IPipelineRunner` in its constructor and a `PipelineContext` at run time.

```csharp
var runner = new TestPipelineRunner(PipelineRunner.Create());
var result = await runner.RunAndGetResultAsync<MyPipeline, EnrichedOrder>(context);
// result is IReadOnlyList<EnrichedOrder> from the sink
```

The pipeline must register an `InMemorySinkNode<TResult>`; otherwise the runner throws.

## Testing Best Practices

1. **Unit test nodes directly** — call `TransformAsync` with controlled inputs and `PipelineContext.CreateDefault()`.
2. **Integration test pipelines** — use `PipelineTestHarness` with in-memory source and sink nodes.
3. **Test error paths** — use `CaptureErrors()` for expected failures.
4. **Parameterize tests** — use `[Theory]` with `[InlineData]` for data-driven node testing.
5. **Mock dependencies** — use FakeItEasy or Moq for external services in nodes.
6. **Name tests clearly** — `MethodName_Condition_ExpectedBehavior`.
