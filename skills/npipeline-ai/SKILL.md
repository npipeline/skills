---
name: npipeline-ai
description: Use when the user wants to add AI/LLM capabilities to NPipeline pipelines using Microsoft.Extensions.AI. Covers AITransformNode, AIEnrichNode, batched and stream variants, AIRouteBuilder, AITransformException, prompt construction, JSON response parsing, and IChatClient integration (OpenAI, Azure OpenAI, Ollama, Anthropic, etc.). Use when user mentions "AI transform", "LLM", "chat client", "AI enrich", "AI route", "OpenAI", "Anthropic", "Ollama", or "IChatClient".
npipelineVersion: "0.52.0"
lastVerified: "2026-06-04"
---

# NPipeline AI Extensions

This skill covers the `NPipeline.Extensions.AI` package — nodes that integrate `Microsoft.Extensions.AI.IChatClient` into pipelines for LLM-powered transforms, enrichment, and routing.

## Package

```
dotnet add package NPipeline.Extensions.AI
```

## Node Variants

The AI extension provides 6 node families across two dimensions:

| | Transform (TIn→TOut) | Enrich (TIn→TIn + field) |
|---|---|---|
| **Per-Item** | `AITransformNode<TIn, TOut>` | `AIEnrichNode<TIn, TField>` |
| **Batched** | `AIBatchedTransformNode<TIn, TOut>` | `AIBatchedEnrichNode<TIn, TField>` |
| **Batched Stream** | `AIBatchedStreamTransformNode<TIn, TOut>` | `AIBatchedStreamEnrichNode<TIn, TField>` |

- **Transform** — Full input-to-output conversion via LLM
- **Enrich** — Augments existing item with an LLM-derived field (less destructive)
- **Per-Item** — One LLM call per item
- **Batched** — Groups items into batches, one LLM call per batch
- **Batched Stream** — Processes whole stream in a single LLM call

## Workflow

### Phase 1: Determine the Node Type

Ask the user: Are you transforming (full conversion) or enriching (adding fields)? Per-item or batched?

| Task | Recommended Node |
|---|---|
| Classify each item into categories | `AITransformNode` (per-item) |
| Add sentiment score to each item | `AIEnrichNode` (per-item) |
| Summarize batches of items | `AIBatchedTransformNode` |
| Add batch-level tags | `AIBatchedEnrichNode` |
| Process entire stream as context | `AIBatchedStreamTransformNode` |

### Phase 2: Register with IChatClient

All AI nodes require an `IChatClient`. Register one for any supported provider:

```csharp
// OpenAI
services.AddChatClient(new OpenAIClient(apiKey).AsChatClient("gpt-4o"));

// Azure OpenAI
services.AddChatClient(new AzureOpenAIClient(endpoint, credential)
    .AsChatClient("deployment-name"));

// Ollama
services.AddChatClient(new OllamaChatClient(new Uri("http://localhost:11434"), "llama3"));

// Anthropic (via Microsoft.Extensions.AI adapter)
services.AddChatClient(/* Anthropic adapter */);
```

### Phase 3: Add the Node to a Pipeline

```csharp
var classification = builder.AddAITransform<ClassifyOrder, Order, Category>(
    chatClient,
    options => options
        .WithPrompt("Classify this order into a category: HighValue, Standard, or LowValue")
        .WithResultMapper((order, category) => order with { Category = category }),
    "classify-orders");
```

Consult `references/ai-nodes-api.md` for the full options builder API, result mapping, routing, and error handling.
