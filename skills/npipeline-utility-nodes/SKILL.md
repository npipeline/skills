---
name: npipeline-utility-nodes
description: Use when the user wants to use NPipeline's built-in utility nodes for common ETL operations. Covers data validation (string, numeric, date, collection), data cleansing, filtering, type conversion, and property enrichment. Use when user mentions "validate data", "cleanse", "filter", "type conversion", "enrich", "data quality", "string validation", "numeric validation", or "deduplicate".
npipelineVersion: "0.67.0"
lastVerified: "2026-09-29"
---

# NPipeline Utility Nodes

This skill covers `NPipeline.Extensions.Nodes` — pre-built nodes for common data processing tasks: validation, cleansing, filtering, conversion, and enrichment.

## Package

```
dotnet add package NPipeline.Extensions.Nodes
```

## Workflow

When the user needs data quality or transformation operations, determine which category they need, then guide them to the appropriate node family.

| User Need | Node Category |
|---|---|
| Check data meets rules | Validation |
| Fix/clean messy data | Cleansing |
| Remove unwanted items | Filtering |
| Change data types | Type Conversion |
| Add computed fields | Enrichment |

See `references/validation-cleansing.md` for the rule tables and per-type method lists.
See `references/filtering-enrichment.md` for filtering, type conversion, and enrichment APIs.

> [!IMPORTANT]
> Rules take a property selector as their first argument, e.g. `IsNotEmpty(o => o.CustomerName)`. There is no `ForProperty(...)` wrapper and no `.OnError(...)` method; error decisions are handled by a node-level error handler (see "Error Behavior" below).

## Quick Examples

### Validation

```csharp
builder.AddStringValidation<Order>(cfg => cfg
    .IsNotEmpty(o => o.CustomerName)
    .HasMaxLength(o => o.CustomerName, 100));

builder.AddNumericValidation<Order>(cfg => cfg
    .IsPositive(o => o.Amount)
    .IsLessThan(o => o.Amount, 10_000));

builder.AddDateTimeValidation<Order>(cfg => cfg
    .IsInPast(o => o.CreatedAt)
    .IsUtc(o => o.CreatedAt));
```

### Cleansing

```csharp
builder.AddStringCleansing<Order>(cfg => cfg
    .Trim(o => o.CustomerName)
    .ToTitleCase(o => o.CustomerName));

builder.AddNumericCleansing<Order>(cfg => cfg
    .Clamp(o => o.Amount, 0m, 10_000m));
```

### Filtering

```csharp
builder.AddFilteringNode<Order>(cfg => cfg
    .Where(o => o.Amount > 0)
    .Where(o => !string.IsNullOrEmpty(o.CustomerName)));
```

### Type Conversion

```csharp
builder.AddTypeConversion<string, int>(cfg => cfg
    .WithConverter(s => int.Parse(s)));
```

### Enrichment

```csharp
builder.AddEnrichment<Order>(cfg => cfg
    .Compute(o => o.TotalAmount, o => o.Amount * o.Quantity));
```

## Error Behavior

Validation, filtering, and conversion nodes throw typed exceptions:

- `ValidationException` — item failed a validation rule
- `FilteringException` — item did not satisfy the filter
- `TypeConversionException` — conversion failed

The `AddXValidation`, `AddFilteringNode`, and `AddTypeConversion` extension methods attach a default error handler (`DefaultValidationErrorHandler<T>`, `DefaultFilteringErrorHandler<T>`, `DefaultTypeConversionErrorHandler<TIn,TOut>`) to the node via `builder.AddResiliencePolicy(handle, handler)`.

- By default that handler returns `Fail`. To skip or dead-letter failing items, pass `applyDefaultErrorHandler: false` and register your own policy, or configure the node's `OnItemFailure` through `builder.WithResilience(...)`.
- The error handler is a node-scoped `IResiliencePolicy`. See the `npipeline-resilience` skill.
