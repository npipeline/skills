# Filtering, Conversion, and Enrichment Reference

## FilteringNode<T>

Removes items that don't satisfy all predicates. Multiple `Where()` calls are combined with AND logic. When an item fails, a `FilteringException` is thrown; a node-level error handler decides whether to Fail (default), Skip, or DeadLetter.

```csharp
builder.AddFilteringNode<Order>(cfg => cfg
    .Where(o => o.Amount > 0)
    .Where(o => !string.IsNullOrEmpty(o.CustomerName)));
```

Give a rejection reason by passing a second argument. `Where` takes either a fixed message or a per-item factory:

```csharp
cfg.Where(o => o.Amount > 0, "Order amount must be positive");
cfg.Where(o => o.Amount > 0, o => $"Order {o.Id} had non-positive amount");
```

> [!TIP]
> To simply drop items without an error decision, use the core `builder.AddFilter(predicate)`, which is a stream transform and buffers nothing. `FilteringNode<T>` is for when a rejected item is an event you want to route.

## TypeConversionNode<TIn, TOut>

Converts between types. Built-in conversions are provided by the `TypeConversions` factory, or you can supply your own delegate.

```csharp
// Custom converter
builder.AddTypeConversion<string, int>(cfg => cfg
    .WithConverter(s => int.Parse(s, CultureInfo.InvariantCulture)));

// Delegate overload (shorthand)
builder.AddTypeConversion<string, decimal>(s => decimal.Parse(s, CultureInfo.InvariantCulture), "string-to-decimal");
```

A node created without a converter fails at runtime; `WithConverter` is required unless the built-in conversion is inferred. On failure, `TypeConversionException` is thrown with the original value and target type.

## EnrichmentNode<T>

Sets or computes fields on existing items. Extends `PropertyTransformationNode<T>`, which uses compiled expression trees for property access.

### Operations

| Operation | Method | Description |
|---|---|---|
| **Compute** | `Compute<TValue>(selector, Func<T,TValue>)` | Compute a field from the item |
| **Lookup** | `Lookup<TKey,TValue>(selector, IReadOnlyDictionary<TKey,TValue>, keySelector)` | Look up a value only when the key exists |
| **Set** | `Set<TKey,TValue>(selector, IReadOnlyDictionary<TKey,TValue>, keySelector)` | Look up a value, defaulting when the key is missing |
| **Default value** | `DefaultValue<TValue>(selector, TValue)` | Set a default when the current value is null/default |

### Example

```csharp
builder.AddEnrichment<Order>(cfg => cfg
    .Compute(o => o.TotalAmount, o => o.Amount * o.Quantity)
    .Set(o => o.CategoryName, categoryNames, o => o.CategoryId));
```

## PropertyAccessor

The enrichment engine compiles property accessors from expression trees at registration time, avoiding reflection in the hot path. It supports nested property paths such as `o => o.Customer.Address.City`.

## PipelineBuilder Extension Methods Summary

All utility nodes are registered through extension methods on `PipelineBuilder`:

```csharp
// Validation
builder.AddStringValidation<T>(cfg => ...)
builder.AddNumericValidation<T>(cfg => ...)
builder.AddDateTimeValidation<T>(cfg => ...)
builder.AddCollectionValidation<T>(cfg => ...)
builder.AddValidationNode<T, TValidationNode>(...)   // custom ValidationNode<T> subclass

// Cleansing
builder.AddStringCleansing<T>(cfg => ...)
builder.AddNumericCleansing<T>(cfg => ...)
builder.AddDateTimeCleansing<T>(cfg => ...)
builder.AddCollectionCleansing<T>(cfg => ...)

// Filtering, conversion, enrichment
builder.AddFilteringNode<T>(cfg => ...)
builder.AddTypeConversion<TIn, TOut>(cfg => ...)
builder.AddEnrichment<T>(cfg => ...)

// Custom property transformation node
builder.AddTransformationNode<T, TTransformationNode>(...)
```

Each `AddXValidation`, `AddFilteringNode`, and `AddTypeConversion` overload has an `applyDefaultErrorHandler` parameter (default `true`). When true, the method attaches the matching `Default…ErrorHandler` policy to the node; pass `false` to register your own or rely on `OnItemFailure`.

The extension methods also have overloads that take no configure delegate and a `string? name`, so a node can be added and configured separately if preferred.
