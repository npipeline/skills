# AI Nodes API Reference

## AITransformNode<TIn, TOut>

Full conversion: item in → LLM → item out. The LLM generates the entire output from the input.

### Registration

```csharp
var handle = builder.AddAITransform<MyTransform, Order, EnrichedOrder>(
    chatClient,
    options => options
        .WithPrompt("Extract key fields from this order...")
        .WithResultMapper((order, enriched) => enriched),
    "ai-transformer");
```

### AITransformOptions<TIn, TOut>

```csharp
var opts = new AITransformOptionsBuilder<Order, Category>()
    .WithPrompt("Classify: {OrderId} {Amount} → category")
    .WithSystemMessage("You are an order classification assistant.")
    .WithTemperature(0.2f)
    .WithMaxTokens(100)
    .WithResultMapper((order, category) => new EnrichedOrder(order, category))
    .Build();
```

### Prompt Construction

Prompts can include template variables from input items. The engine replaces `{PropertyName}` with the item's property values.

## AIEnrichNode<TIn, TField>

Augments an existing item with one LLM-derived field. Less destructive than full transform — the original item is preserved.

### Registration

```csharp
var handle = builder.AddAIEnrich<Order, string>(
    chatClient,
    options => options
        .WithPrompt("Summarize this order in one sentence...")
        .WithResultMapper((order, summary) => order with { Summary = summary }),
    "summarize-order");
```

## Batched Variants

### AIBatchedTransformNode<TIn, TOut>

Sends batches to the LLM. The transform receives `IReadOnlyList<Order>` and the prompt describes the batch:

```csharp
var handle = builder.AddAIBatchedTransform<MyBatcher, Order, Category>(
    chatClient,
    options => options
        .WithPrompt("Classify each order in this batch. Return a JSON array of categories.")
        .WithBatchSize(50),
    "batch-classify");
```

### AIBatchedEnrichNode<TIn, TField>

Batch enrichment — enriches each item in a batch with an LLM-derived field.

### Stream Variants

`AIBatchedStreamTransformNode<TIn, TOut>` and `AIBatchedStreamEnrichNode<TIn, TField>` process the entire stream in one LLM call.

## AIRouteBuilder<T>

Conditional LLM-powered routing — the LLM decides where each item goes:

```csharp
builder.AddAIRoute<TIn, TField>(chatClient, route => route
    .WithRoute("priority", "Is this a high-priority order?", RouteMatchMode.FirstMatch)
    .WithRoute("regular", "Is this a regular order?", RouteMatchMode.FirstMatch)
    .WithOtherwise("unclassified"));
```

Two route builder methods available: `AddAIRoute<TIn, TField>` for per-item routing and `AddAIBatchedStreamRoute<TIn, TField>` for batched stream routing.

## AIInvoker

The internal engine handles LLM invocation with retry, response sanitization, JSON deserialization with graceful fallback, and batch count mismatch detection with automatic retry. This is an internal implementation detail — you don't interact with it directly.

## AITransformException

When an AI node fails, the exception carries rich context:

```csharp
public sealed class AITransformException : PipelineException
{
    string ErrorCode              // Error code (e.g., "AI_TRANSFORM_ERROR")
    object? OriginalItem          // The item being processed
    string? PromptSent            // The prompt that was sent
    string? ModelUsed             // Which model responded
    string? RawResponse           // The raw LLM response (for debugging)
}
```

## Error Handling

AI nodes throw `AITransformException` on failure. Handle these in resilience policies:

```csharp
public override Task<ResilienceDecision> DecideItemFailureAsync<TIn, TOut>(...)
{
    if (exception is AITransformException aiEx)
    {
        _logger.LogWarning("AI failed for item {Item}: {Response}",
            aiEx.OriginalItem, aiEx.RawResponse);
        return Task.FromResult(ResilienceDecision.Skip);
    }
    return base.DecideItemFailureAsync<TIn, TOut>(...);
}
```

## Best Practices

1. **Use batched modes for cost efficiency** — One LLM call per batch of 50 items is cheaper than 50 individual calls.
2. **Set temperature low** (0.0–0.3) for classification/categorization tasks.
3. **Use enrichment instead of transform** when adding fields — preserves existing data.
4. **Handle AI failures gracefully** — LLMs can produce unexpected output. Use resilience policies with `Skip` or `DeadLetter` decisions for AI exceptions.
5. **Template prompts carefully** — Use `{PropertyName}` syntax for item data injection.
