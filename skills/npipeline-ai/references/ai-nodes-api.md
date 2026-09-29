# AI Nodes API Reference

## NPipeline.Extensions.AI.Chat

### ChatTransformNode<TIn, TOut>

Full conversion: item in, model response deserialized to `TOut`.

```csharp
using NPipeline.Extensions.AI.Chat;

var handle = builder.AddChatTransform<Comment, ClassificationResult>(chatClient, options => options
    .WithSystemPrompt("Classify the comment as Greeting, Question, Complaint, or Spam.")
    .WithItemTemplate(comment => $"Text: {comment.Text}")
    .WithNativeStructuredOutput()
    .WithTemperature(0.1f)
    .WithMaxOutputTokens(128));
```

### ChatTransformOptions<TIn, TOut>

```csharp
public sealed record ChatTransformOptions<TIn, TOut>(
    string? SystemPrompt = null,             // Required
    Func<TIn, string>? ItemTemplate = null,  // Required
    float? Temperature = null,
    int? MaxOutputTokens = null,
    bool UseNativeStructuredOutput = false,
    Action<ChatOptions>? ConfigureOptions = null);
```

Builder methods: `WithSystemPrompt`, `WithItemTemplate`, `WithTemperature`, `WithMaxOutputTokens`, `WithNativeStructuredOutput`, `WithConfigureOptions`. `Build()` throws `InvalidOperationException` when the system prompt or item template is missing.

### ChatEnrichmentNode<TIn, TField>

Augments the original item with a model-derived field. Registered with `AddChatEnrichment<TIn,TField>` and returns a `TransformNodeHandle<TIn,TIn>`.

```csharp
var handle = builder.AddChatEnrichment<Article, SummaryResult>(chatClient, options => options
    .WithSystemPrompt("Summarize the article in one sentence.")
    .WithItemTemplate(article => article.Body)
    .WithResultMapper((article, result) => article with { Summary = result.Summary }));
```

`ResultMapper<TIn,TField>` has the signature `TIn (TIn input, TField result)`.

### Batched Variants

| Method | Node | Input → Output |
|---|---|---|
| `AddChatBatchedTransform<TIn,TOut>` | `ChatBatchedTransformNode<TIn,TOut>` | `IReadOnlyCollection<TIn> → IReadOnlyCollection<TOut>` |
| `AddChatBatchedEnrichment<TIn,TField>` | `ChatBatchedEnrichmentNode<TIn,TField>` | `IReadOnlyCollection<TIn> → IReadOnlyCollection<TIn>` |
| `AddChatBatchedStreamTransform<TIn,TOut>` | `ChatBatchedStreamTransformNode<TIn,TOut>` | `TIn → TOut` |
| `AddChatBatchedStreamEnrichment<TIn,TField>` | `ChatBatchedStreamEnrichmentNode<TIn,TField>` | `TIn → TIn` |

Batched transforms use `WithBatchTemplate(Func<IReadOnlyCollection<TIn>, string>)`. The model must return one result per input item (an object with an `Items` array). Stream-batched variants add `WithBatchSize(int)` and `WithBatchTimeout(TimeSpan)`: they buffer up to the batch size, flush an incomplete batch after the timeout, make one request, and emit individual results.

`AddChatBatchedEnrichmentWithUnbatch<T,TField>(chatClient, batchSize, batchTimeout, configure, name?)` builds a batcher, a batched enrichment node, and an unbatcher, returning `(inputHandle, outputHandle)` to connect as a single `T → T` stage.

### Exceptions

`ChatTransformException` is raised when a chat node fails (invocation or deserialization).

## NPipeline.Extensions.AI.Decisions

### IAIClassifier<TInput, TLabel>

```csharp
public interface IAIClassifier<in TInput, TLabel>
    where TLabel : notnull
{
    ValueTask<AIClassification<TLabel>> ClassifyAsync(
        TInput input,
        CancellationToken cancellationToken = default);
}
```

### AIClassification<TLabel>

```csharp
public sealed record AIClassification<TLabel>(
    TLabel Label,
    double Confidence,
    IReadOnlyDictionary<TLabel, double> Probabilities,
    AIInvocationMetadata Metadata)
    where TLabel : notnull;
```

`AIInvocationMetadata` carries `Provider`, `Model`, `RequestId`, and optional `AIUsage` (input/output tokens).

### AddAIRoute<TInput, TLabel>

Adds an `AIClassificationNode<TInput,TLabel>` followed by a confidence-aware route, and returns an `AIRouteBuilder<TInput,TLabel>` that implements `IInputNodeHandle<TInput>`.

```csharp
var route = builder.AddAIRoute<Ticket, TicketRoute>(classifier, "ticket-route")
    .WhenLabel(TicketRoute.Billing, billingSink, minimumConfidence: 0.75)
    .Otherwise(reviewSink);

builder.Connect(source, route);
```

| Method | Behavior |
|---|---|
| `WhenLabel(label, target, minimumConfidence = 0)` | Selected label matches and confidence meets the threshold |
| `WhenProbability(label, minimumProbability, target)` | Any label's probability meets the threshold (including a nonwinning label) |
| `When(Func<AIClassification<TLabel>, bool> predicate, target)` | Custom predicate over the full classification |
| `Otherwise(target)` | Fallback for items matching no branch (one only) |
| `WithMatchMode(RouteMatchMode)` | `FirstMatch` (default) or `AllMatches` |

Advanced handles: `ClassificationHandle` (sources `AIClassifiedItem<TInput,TLabel>` right after classification) and `RouteHandle`.

`AIClassifiedItem<TInput, TLabel>` is the envelope `(TInput Item, AIClassification<TLabel> Classification)`.

## NPipeline.Extensions.AI.Decisions.Jev

### AddJevRoute<TInput, TLabel>

```csharp
var route = builder.AddJevRoute<Ticket, TicketRoute>(jevClient, options => options
        .WithState(ticket => new { ticket.Subject, ticket.Message, ticket.CustomerTier })
        .WithInstructions("Which team should handle this ticket?")
        .AddChoice(TicketRoute.Billing, "billing", "Charges, invoices, and refunds")
        .AddChoice(TicketRoute.Technical, "technical", "Bugs, outages, and integrations")
        .AddChoice(TicketRoute.Other, "other", "None of the other routes apply"))
    .WhenLabel(TicketRoute.Billing, billingSink, minimumConfidence: 0.75)
    .Otherwise(reviewSink);
```

### JevChoiceClassifierOptionsBuilder<TInput, TLabel>

| Method | Required | Description |
|---|---|---|
| `WithState(Func<TInput, object?>)` | Yes | Projects the item into string or structured JSON state (return `JsonNode` for direct control) |
| `WithInstructions(string)` / `WithInstructions(JsonNode)` | Yes | Text or structured instructions for the Choice question |
| `AddChoice(label, wireName, description?)` / `AddChoice(label, wireName, JsonNode?)` | At least two | Maps a typed label to the provider wire name and criteria |
| `WithModel(string)` | No | Overrides the client's default model |
| `WithQuestionId(string)` | No | Answer correlation key; default `route` |

### JevClient and Options

```csharp
public sealed class JevClientOptions
{
    public string ApiKey { get; init; }                                  // Required
    public Uri? BaseUri { get; init; }                                   // Default https://api.typesafe.ai
    public string? DefaultModel { get; init; }                           // Default jev-latest
    public TimeSpan AttemptTimeout { get; init; } = TimeSpan.FromSeconds(10);
    public JevRetryPolicy Retry { get; init; } = new();
    public TimeProvider TimeProvider { get; init; } = TimeProvider.System;
}
```

`JevClientOptions.Retry` (`JevRetryPolicy`) defaults to `MaxRetries = 2`, exponential backoff with jitter. Set `MaxRetries = 0` when an NPipeline resilience policy owns retries, to avoid multiplying attempts.

`JevChoiceClassifier<TInput,TLabel>` returns `AIClassification<TLabel>` with `Provider = "typesafe"`, the concrete response model, request ID, and token usage. It rejects responses that omit the configured answer, return the wrong type, select an unknown wire name, or omit a configured probability.

### Direct System One Evaluation

`IJevClient.EvaluateAsync(state, questions, model?, cancellationToken?)` evaluates several independent questions about one state in a single request. `JevChoiceQuestion`/`JevChoiceAnswer`, `JevScoreQuestion`/`JevScoreAnswer`, and `JevNoulQuestion`/`JevNoulAnswer` cover choice, score, and yes/no judgments.

### Exceptions

`JevApiException` (derived from `AIInvocationException`) is raised for nonretryable or exhausted responses and carries `StatusCode`, `ResponseBody`, `Headers`, `Provider`, `Model`, and `RequestId` without exposing the API key.
