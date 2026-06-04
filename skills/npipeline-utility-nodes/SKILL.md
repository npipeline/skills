---
name: npipeline-utility-nodes
description: Use when the user wants to use NPipeline's built-in utility nodes for common ETL operations. Covers data validation (string, numeric, date, collection), data cleansing, filtering, type conversion, and property enrichment. Use when user mentions "validate data", "cleanse", "filter", "type conversion", "enrich", "data quality", "string validation", "numeric validation", or "deduplicate".
npipelineVersion: "0.52.0"
lastVerified: "2026-06-04"
---

# NPipeline Utility Nodes

This skill covers `NPipeline.Extensions.Nodes` — a library of pre-built nodes for common data processing tasks: validation, cleansing, filtering, conversion, and enrichment.

## Package

```
dotnet add package NPipeline.Extensions.Nodes
```

## Workflow

When the user needs data quality or transformation operations, determine which category of operation they need, then guide them to the appropriate node family.

| User Need | Node Category |
|---|---|
| Check data meets rules | Validation |
| Fix/clean messy data | Cleansing |
| Remove unwanted items | Filtering |
| Change data types | Type Conversion |
| Add computed fields | Enrichment |

See `references/validation-cleansing.md` for validation rules, cleansing operations, and per-type tables.
See `references/filtering-enrichment.md` for filtering, type conversion, and enrichment APIs.

## Quick Examples

### Validation

```csharp
builder.AddStringValidation<Order>(cfg => cfg
    .ForProperty(o => o.CustomerName)
    .IsNotEmpty()
    .HasMaxLength(100));

builder.AddNumericValidation<Order>(cfg => cfg
    .ForProperty(o => o.Amount)
    .IsPositive()
    .IsLessThan(10000));

builder.AddDateTimeValidation<Order>(cfg => cfg
    .ForProperty(o => o.CreatedAt)
    .IsNotInFuture()
    .IsUtc());
```

### Cleansing

```csharp
builder.AddStringCleansing<Order>(cfg => cfg
    .ForProperty(o => o.CustomerName)
    .Trim()
    .ToTitleCase());

builder.AddNumericCleansing<Order>(cfg => cfg
    .ForProperty(o => o.Amount)
    .Clamp(0, 10000));
```

### Filtering

```csharp
builder.AddFilteringNode<Order>(cfg => cfg
    .Where(o => o.Amount > 0)
    .Where(o => !string.IsNullOrEmpty(o.CustomerName)));
```

### Type Conversion

```csharp
builder.AddTypeConversion<string, int>("string-to-int");
```

### Enrichment

```csharp
builder.AddEnrichment<Order>(cfg => cfg
    .Set(o => o.ProcessedAt, DateTime.UtcNow));
```

## Error Behavior

Validation and filtering nodes throw typed exceptions with configurable resilience decisions:

- `ValidationException` — Item failed validation rules
- `FilteringException` — Item was filtered out
- `TypeConversionException` — Conversion failed

Default handlers return `Fail`. Override to `Skip` or `DeadLetter`:

```csharp
builder.AddStringValidation<Order>(cfg => cfg
    .ForProperty(o => o.CustomerName)
    .IsNotEmpty()
    .OnError(ResilienceDecision.Skip)); // Skip items that fail validation
```
