---
name: npipeline-ai
description: Use when the user wants to add AI/LLM capabilities to NPipeline pipelines. Covers NPipeline.Extensions.AI.Chat transform and enrichment nodes (per-item, batched, batched-stream) over Microsoft.Extensions.AI.IChatClient, and NPipeline.Extensions.AI.Decisions typed classification and confidence-aware routing, including TypeSafe Jev via AddJevRoute. Use when user mentions "AI transform", "LLM", "chat client", "AI enrich", "AI route", "classification", "Jev", "TypeSafe", "OpenAI", "Anthropic", "Ollama", or "IChatClient".
npipelineVersion: "0.67.0"
lastVerified: "2026-09-29"
---

# NPipeline AI Extensions

NPipeline's AI support is split across packages: chat-based transforms and enrichment live in `NPipeline.Extensions.AI.Chat`; typed classification and routing live in `NPipeline.Extensions.AI.Decisions`, with a TypeSafe Jev provider in `NPipeline.Extensions.AI.Decisions.Jev`. A shared `NPipeline.Extensions.AI` package holds common contracts.

## Packages

```
dotnet add package NPipeline.Extensions.AI.Chat
dotnet add package NPipeline.Extensions.AI.Decisions
dotnet add package NPipeline.Extensions.AI.Decisions.Jev   # optional, TypeSafe Jev
```

> [!IMPORTANT]
> Chat nodes are `ChatTransformNode`, `ChatEnrichmentNode`, and their batched variants, registered with `AddChatTransform`, `AddChatEnrichment`, `AddChatBatchedTransform`, `AddChatBatchedStreamTransform`, `AddChatBatchedEnrichment`, and `AddChatBatchedEnrichmentWithUnbatch`. Names such as `AITransformNode`, `AddAITransform`, and `AITransformOptionsBuilder` are not part of this package.

## Chat Nodes (NPipeline.Extensions.AI.Chat)

The chat package turns items into chat requests through any `Microsoft.Extensions.AI.IChatClient` and deserializes the response into a strongly typed value.

### Node Variants

| Scenario | Method | Input → Output |
|---|---|---|
| Transform one item per request | `AddChatTransform<TIn,TOut>` | `TIn → TOut` |
| Transform a collection in one request | `AddChatBatchedTransform<TIn,TOut>` | `IReadOnlyCollection<TIn> → IReadOnlyCollection<TOut>` |
| Buffer a stream, transform each batch, emit individual results | `AddChatBatchedStreamTransform<TIn,TOut>` | `TIn → TOut` |
| Enrich one item per request | `AddChatEnrichment<TIn,TField>` | `TIn → TIn` |
| Enrich a collection in one request | `AddChatBatchedEnrichment<TIn,TField>` | `IReadOnlyCollection<TIn> → IReadOnlyCollection<TIn>` |
| Buffer a stream, enrich each batch, emit individual items | `AddChatBatchedStreamEnrichment<TIn,TField>` | `TIn → TIn` |
| Batch, enrich, unbatch as one chain | `AddChatBatchedEnrichmentWithUnbatch<T,TField>` | `T → T` connection handles |

The node classes are `ChatTransformNode<TIn,TOut>`, `ChatEnrichmentNode<TIn,TField>`, `ChatBatchedTransformNode<TIn,TOut>`, `ChatBatchedStreamTransformNode<TIn,TOut>`, `ChatBatchedEnrichmentNode<TIn,TField>`, and `ChatBatchedStreamEnrichmentNode<TIn,TField>`.

### Register a Chat Client

```csharp
// OpenAI
services.AddChatClient(new OpenAIClient(apiKey).AsChatClient("gpt-4o"));

// Azure OpenAI
services.AddChatClient(new AzureOpenAIClient(endpoint, credential).AsChatClient("deployment-name"));

// Ollama
services.AddChatClient(new OllamaChatClient(new Uri("http://localhost:11434"), "llama3"));
```

### Transform

```csharp
using NPipeline.Extensions.AI.Chat;

public record Comment(string Text, string Author);
public record ClassificationResult(string Category, double Confidence);

var classify = builder.AddChatTransform<Comment, ClassificationResult>(chatClient, options => options
    .WithSystemPrompt("Classify the comment as Greeting, Question, Complaint, or Spam.")
    .WithItemTemplate(comment => $"Text: {comment.Text}")
    .WithNativeStructuredOutput()
    .WithTemperature(0.1f)
    .WithMaxOutputTokens(128));
```

### Enrich

`AddChatEnrichment` maps the model result back into the original item through a `ResultMapper<TIn,TField>` with signature `TIn (TIn input, TField result)`:

```csharp
var enrich = builder.AddChatEnrichment<Article, SummaryResult>(chatClient, options => options
    .WithSystemPrompt("Summarize the article in one sentence.")
    .WithItemTemplate(article => article.Body)
    .WithResultMapper((article, result) => article with { Summary = result.Summary })
    .WithMaxOutputTokens(128));
```

### Batched and Batch/Unbatch Variants

```csharp
// Pre-batched: input is already an IReadOnlyCollection<TIn>
var classifyBatch = builder.AddChatBatchedTransform<Comment, ClassificationResult>(
    chatClient,
    options => options
        .WithSystemPrompt("Classify every comment. Return an Items array in input order.")
        .WithBatchTemplate(batch => string.Join("\n", batch.Select((c, i) => $"{i + 1}. {c.Text}")))
        .WithNativeStructuredOutput());

// Stream-batched: individual item types at the boundary, internal batching
var classifyStream = builder.AddChatBatchedStreamTransform<Comment, ClassificationResult>(
    chatClient,
    options => options
        .WithSystemPrompt("Classify every comment. Return an Items array in input order.")
        .WithBatchTemplate(batch => string.Join("\n", batch.Select((c, i) => $"{i + 1}. {c.Text}")))
        .WithBatchSize(32)
        .WithBatchTimeout(TimeSpan.FromSeconds(2)));
```

### Execution Model

1. A template delegate formats an item or batch as the user message.
2. The node sends a system message and the formatted user message through `IChatClient`.
3. The node removes a surrounding Markdown code fence when the model includes one.
4. `System.Text.Json` deserializes the response into the configured output type.
5. The typed result flows on, or an enrichment mapper merges it into the input.

Calls are stateless: nodes don't retain conversation history between items or batches. A batched transform must return one result per input item (an object with an `Items` array). A deserialization or invocation failure raises `ChatTransformException`.

Consult `references/ai-nodes-api.md` for the options builders and more detail.

## Decision Nodes (NPipeline.Extensions.AI.Decisions)

The decisions package separates the decision from the domain item: a classifier produces a typed decision with confidence and probabilities, and a confidence-aware route sends the original item type to a target.

### Classifier Contract

```csharp
public interface IAIClassifier<in TInput, TLabel>
    where TLabel : notnull
{
    ValueTask<AIClassification<TLabel>> ClassifyAsync(
        TInput input,
        CancellationToken cancellationToken = default);
}

public sealed record AIClassification<TLabel>(
    TLabel Label,
    double Confidence,
    IReadOnlyDictionary<TLabel, double> Probabilities,
    AIInvocationMetadata Metadata)
    where TLabel : notnull;
```

Implement `IAIClassifier<TInput,TLabel>` directly, or use a provider adapter such as Jev.

### Route Decisions

```csharp
var route = builder.AddAIRoute<Ticket, TicketRoute>(classifier, "ticket-route")
    .WhenLabel(TicketRoute.Billing, billingSink, minimumConfidence: 0.75)
    .WhenLabel(TicketRoute.Technical, technicalSink, minimumConfidence: 0.65)
    .Otherwise(reviewSink);

builder.Connect(source, route);
```

`AIRouteBuilder<TInput,TLabel>` implements `IInputNodeHandle<TInput>`, so connect the upstream node directly to it. Route methods:

| Method | Behavior |
|---|---|
| `WhenLabel(label, target, minimumConfidence)` | Matches the selected label when confidence meets the threshold |
| `WhenProbability(label, minimumProbability, target)` | Matches any label whose probability meets the threshold |
| `When(predicate, target)` | Predicate over the complete `AIClassification<TLabel>` |
| `Otherwise(target)` | Receives items that matched no branch (one fallback only) |
| `WithMatchMode(mode)` | `RouteMatchMode.FirstMatch` (default) or `AllMatches` |

Confidence and probability thresholds must be finite values in the range `0`–`1`. Internally `AIClassificationNode<TInput,TLabel>` wraps the item in an `AIClassifiedItem<TInput,TLabel>` envelope; each route branch passes through an internal unwrap node so targets stay typed as `TInput`.

### TypeSafe Jev (NPipeline.Extensions.AI.Decisions.Jev)

```csharp
using NPipeline.Extensions.AI.Decisions.Jev;

var jevClient = new JevClient(httpClient, new JevClientOptions { ApiKey = apiKey });

var route = builder.AddJevRoute<Ticket, TicketRoute>(jevClient, options => options
        .WithState(ticket => new { ticket.Subject, ticket.Message, ticket.CustomerTier })
        .WithInstructions("Which team should handle this ticket?")
        .AddChoice(TicketRoute.Billing, "billing", "Charges, invoices, and refunds")
        .AddChoice(TicketRoute.Technical, "technical", "Bugs, outages, and integrations")
        .AddChoice(TicketRoute.Other, "other", "None of the other routes apply"))
    .WhenLabel(TicketRoute.Billing, billingSink, minimumConfidence: 0.75)
    .WhenLabel(TicketRoute.Technical, technicalSink, minimumConfidence: 0.65)
    .Otherwise(reviewSink);
```

`JevChoiceClassifier<TInput,TLabel>` returns `AIClassification<TLabel>` with the selected label, Jev's confidence, a probability per configured choice, and metadata with `Provider` set to `typesafe`. Use `IJevClient.EvaluateAsync` directly for several independent judgments about one state, including `JevScoreQuestion` and `JevNoulQuestion`. Errors raise `JevApiException` (derived from `AIInvocationException`).

Consult `references/ai-nodes-api.md` for the Jev options and error surface.

## Best Practices

1. **Use batched modes for cost efficiency** — one request per batch of items costs less than one per item.
2. **Set temperature low** (0.0–0.3) for classification and extraction.
3. **Use enrichment instead of transform** when adding fields, to preserve existing data.
4. **Handle failures with a resilience policy** — see the `npipeline-resilience` skill. Classifier and chat failures propagate so policies can apply `Skip` or `DeadLetter`.
5. **Calibrate confidence thresholds** against representative data; there is no universal threshold.
