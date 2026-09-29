---
name: npipeline-testing
description: Use when the user wants to test NPipeline nodes or pipelines. Covers unit testing individual nodes, integration testing with PipelineTestHarness, InMemorySourceNode/InMemorySinkNode, error path testing with CaptureErrors(), assertion helpers (FluentAssertions and AwesomeAssertions), and test runner utilities. Use when user mentions "test my pipeline", "unit test", "integration test", "PipelineTestHarness", "mock node", or "assert".
npipelineVersion: "0.67.0"
lastVerified: "2026-09-29"
---

# NPipeline Testing

This skill covers testing NPipeline nodes and pipelines: unit tests for individual nodes, integration tests for full pipelines, error path testing, and assertion helpers.

## Workflow

When the user wants to test NPipeline code, determine the testing scope (unit or integration), then guide them through the appropriate test utilities.

### Phase 1: Unit Testing Individual Nodes

Test nodes in isolation by calling their methods directly. Use `PipelineContext.CreateDefault()` for a context.

```csharp
[Fact]
public async Task ValidateOrder_MissingName_ThrowsValidationException()
{
    var node = new ValidateOrder();
    var order = new Order { CustomerName = "" };
    var context = PipelineContext.CreateDefault();

    await Assert.ThrowsAsync<ValidationException>(
        async () => await node.TransformAsync(order, context, CancellationToken.None));
}

[Fact]
public async Task ValidateOrder_ValidInput_ReturnsValidatedOrder()
{
    var node = new ValidateOrder();
    var order = new Order { CustomerName = "Acme Corp" };
    var context = PipelineContext.CreateDefault();

    var result = await node.TransformAsync(order, context, CancellationToken.None);

    result.ValidatedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(1));
}
```

> [!IMPORTANT]
> `TransformAsync` returns `ValueTask<TOut>`. `Assert.ThrowsAsync` needs an `async` lambda that awaits the `ValueTask` (`async () => await node.TransformAsync(...)`). `PipelineContext.Default` does not exist; use `PipelineContext.CreateDefault()`.

### Phase 2: Integration Testing with PipelineTestHarness

Test complete pipeline definitions end-to-end:

```csharp
[Fact]
public async Task OrderPipeline_ValidOrders_AllSucceed()
{
    var orders = new[] { new Order { Id = 1 }, new Order { Id = 2 } };
    var harvester = new InMemorySinkNode<EnrichedOrder>();

    var result = await new PipelineTestHarness<OrderPipeline>()
        .WithParameter("input", orders)
        .WithContextItem("harvest", harvester)
        .RunAsync();

    result.AssertSuccess()
        .AssertNoErrors()
        .AssertCompletedWithin(TimeSpan.FromSeconds(10));

    var saved = harvester.Items;
    saved.Should().HaveCount(2);
}
```

### Phase 3: Error Path Testing

Use `CaptureErrors()` to capture exceptions without the harness throwing:

```csharp
[Fact]
public async Task Pipeline_TransientError_RetriesAndSucceeds()
{
    var result = await new PipelineTestHarness<MyPipeline>()
        .CaptureErrors(ResilienceDecision.Retry)  // Retry on errors, but still capture
        .RunAsync();

    result.Errors.Should().BeEmpty();
    result.AssertSuccess();
}
```

`CaptureErrors()` takes an optional `ResilienceDecision` (default `Skip`). It wraps whichever policy the run resolves, so a pipeline's own policy still runs first.

### Phase 4: Assertion Helpers

Two assertion libraries are available (`NPipeline.Extensions.Testing.FluentAssertions` and `.AwesomeAssertions`):

```csharp
result.AssertSuccess().AssertNoErrors().AssertErrorCount(0);
result.GetSink<EnrichedOrder>().Items.Should().HaveCount(10);
```

Consult `references/testing-api.md` for the complete test harness and assertion APIs.

## Next Steps

- For in-memory test nodes and the harness API, see `references/testing-api.md`.
- For resilience behavior under test, see the `npipeline-resilience` skill.
