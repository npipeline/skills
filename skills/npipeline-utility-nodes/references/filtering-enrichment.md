# Filtering, Conversion, and Enrichment Reference

## FilteringNode<T>

Removes items that don't match predicates. Multiple `Where()` calls are combined with AND logic:

```csharp
builder.AddFilteringNode<Order>(cfg => cfg
    .Where(o => o.Amount > 0)
    .Where(o => !string.IsNullOrEmpty(o.CustomerName))
    .WithReason("Order is empty or has zero amount"));
```

When an item fails all predicates, a `FilteringException` is thrown. Configure the decision:

```csharp
cfg.Where(o => o.Amount > 0)
   .OnError(ResilienceDecision.Skip);
```

## TypeConversionNode<TIn, TOut>

Converts data types between pipeline stages. Built-in conversions via `TypeConversions` factory:

| From | To |
|---|---|
| `string` | `int`, `double`, `decimal`, `bool`, `DateTime`, `Enum` |
| `int` | `string`, `double`, `decimal` |
| `double` | `string`, `int`, `decimal` |
| `DateTime` | `string` |
| ... and more | |

### Usage

```csharp
// Built-in conversion
builder.AddTypeConversion<string, decimal>("string-to-decimal");

// Custom conversion
public class MyConverter : TypeConversionNode<Order, OrderDto>
{
    public override Task<OrderDto> TransformAsync(
        Order item, PipelineContext ctx, CancellationToken ct)
    {
        return Task.FromResult(new OrderDto
        {
            Id = item.Id,
            Total = item.Amount * item.Quantity
        });
    }
}
```

When conversion fails, `TypeConversionException` is thrown with the original value and target type.

## EnrichmentNode<T>

Adds or computes fields on existing items. Extends `PropertyTransformationNode<T>` which uses compiled expression trees for zero-allocation property access.

### Operation Types

| Operation | Method | Description |
|---|---|---|
| **Lookup** | `Lookup<TField>(Func<T, TField>)` | Resolve value from a key-value lookup |
| **Set** | `Set<TField>(Expr, TField)` | Set a property to a fixed value |
| **Compute** | `Compute<TField>(Expr, Func<T, TField>)` | Compute a field from the item |
| **Default from lookup** | `DefaultFromLookup<TField>(...)` | Lookup with fallback |
| **Default value** | `DefaultValue<TField>(Expr, TField)` | Set default if current value is null/default |

### Example

```csharp
builder.AddEnrichment<Order>(cfg => cfg
    .Set(o => o.ProcessedAt, DateTime.UtcNow)
    .Compute(o => o.TotalAmount, o => o.Amount * o.Quantity)
    .Lookup(o => o.Category, categoryResolver));
```

## PropertyAccessor

The enrichment engine uses compiled expression trees for property access, avoiding reflection overhead:

```csharp
// Expression tree compiled to delegate at registration time
Expression<Func<T, TProperty>> expr = o => o.CustomerName;
// Compiled into:
Func<T, TProperty> getter = ...;
Action<T, TProperty> setter = ...;
```

Supports nested property paths (e.g., `o => o.Customer.Address.City`).

## PipelineBuilder Extension Methods Summary

All utility nodes are registered through extension methods on `PipelineBuilder`:

```csharp
// Validation
builder.AddStringValidation<T>()
builder.AddNumericValidation<T>()
builder.AddDateTimeValidation<T>()
builder.AddCollectionValidation<T>()
builder.AddValidationNode<T>()  // Generic custom validation

// Cleansing
builder.AddStringCleansing<T>()
builder.AddNumericCleansing<T>()
builder.AddDateTimeCleansing<T>()
builder.AddCollectionCleansing<T>()

// Conversion and Filtering
builder.AddFilteringNode<T>()
builder.AddTypeConversion<TIn, TOut>()

// Enrichment
builder.AddEnrichment<T>()

// Custom
builder.AddTransformationNode<T, TNode>()
```
