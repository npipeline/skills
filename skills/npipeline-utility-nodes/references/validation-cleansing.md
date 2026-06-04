# Validation and Cleansing Reference

## StringValidationNode<T> — 12 Rules

| Rule | Method | Description |
|---|---|---|
| Not empty | `IsNotEmpty()` | Fails if null or "" |
| Not whitespace | `IsNotWhitespace()` | Fails if null, "", or whitespace-only |
| Min length | `HasMinLength(int)` | Min character count |
| Max length | `HasMaxLength(int)` | Max character count |
| Is email | `IsEmail()` | Validates email format |
| Is URL | `IsUrl()` | Validates URL format |
| Is GUID | `IsGuid()` | Validates GUID format |
| Is alphanumeric | `IsAlphanumeric()` | Only letters and digits |
| Matches regex | `Matches(string)` | Custom regex pattern |
| ... and more | | |

### Example

```csharp
builder.AddStringValidation<Order>(cfg => cfg
    .ForProperty(o => o.Email)
    .IsNotEmpty()
    .IsEmail()
    .HasMaxLength(255));
```

Multiple properties can be validated simultaneously by chaining `.ForProperty()` calls.

## NumericValidationNode<T> — 20+ Rules

Supports `int`, `double`, `decimal`, and their nullable variants.

| Rule | Method |
|---|---|
| Not zero | `IsNotZero()` |
| Positive | `IsPositive()` |
| Negative | `IsNegative()` |
| Greater than | `IsGreaterThan(T)` |
| Less than | `IsLessThan(T)` |
| In range | `IsInRange(T, T)` |
| Is integer | `IsInteger()` (for floating types) |
| ... and more | |

### Example

```csharp
builder.AddNumericValidation<Order>(cfg => cfg
    .ForProperty(o => o.Amount)
    .IsPositive()
    .IsLessThan(100000)
    .ForProperty(o => o.Quantity)
    .IsGreaterThan(0)
    .IsInteger());
```

## DateTimeValidationNode<T> — 15+ Rules

| Rule | Method |
|---|---|
| In future | `IsInFuture()` |
| In past | `IsInPast()` |
| Not in future | `IsNotInFuture()` |
| Is UTC | `IsUtc()` |
| Is weekday | `IsWeekday()` |
| Is weekend | `IsWeekend()` |
| Is day of week | `IsDayOfWeek(DayOfWeek)` |
| In range | `IsInRange(DateTime, DateTime)` |
| ... and more | |

### Example

```csharp
builder.AddDateTimeValidation<Order>(cfg => cfg
    .ForProperty(o => o.DeliveryDate)
    .IsInFuture()
    .IsWeekday()
    .ForProperty(o => o.CreatedAt)
    .IsNotInFuture());
```

## CollectionValidationNode<T> — 10 Rules

| Rule | Method |
|---|---|
| Not empty | `IsNotEmpty()` |
| Min count | `HasMinCount(int)` |
| Max count | `HasMaxCount(int)` |
| All match | `AllMatch(Func<T, bool>)` |
| All unique | `AllUnique()` |
| Contains | `Contains(T)` |
| ... and more | |

## StringCleansingNode<T> — 14 Operations

| Operation | Method | Description |
|---|---|---|
| Trim | `Trim()` | Removes leading/trailing whitespace |
| Collapse whitespace | `CollapseWhitespace()` | Multiple spaces → single space |
| To title case | `ToTitleCase()` | "john smith" → "John Smith" |
| To upper | `ToUpper()` | Uppercase conversion |
| To lower | `ToLower()` | Lowercase conversion |
| Truncate | `Truncate(int)` | Cut to max length |
| Replace | `Replace(string, string)` | Substring replacement |
| Default if null | `DefaultIfNullOrWhitespace(string)` | Fallback value |
| ... and more | | |

## NumericCleansingNode<T>

| Operation | Method |
|---|---|
| Clamp | `Clamp(T, T)` — constrain to range |
| Min | `Min(T)` — floor value |
| Max | `Max(T)` — ceiling value |
| Round | `Round(int)` — decimal places |
| Floor | `Floor()` |
| Ceiling | `Ceiling()` |
| Absolute value | `AbsoluteValue()` |
| Scale | `Scale(double)` — multiply by factor |

## DateTimeCleansingNode<T>

| Operation | Method |
|---|---|
| Specify kind | `SpecifyKind(DateTimeKind)` |
| To UTC | `ToUtc()` |
| To local | `ToLocal()` |
| Strip time | `StripTime()` — keep date only |
| Truncate to minute | `Truncate(TimeSpan)` |
| Round to nearest minute | `RoundToMinute()` |
| Clamp | `Clamp(DateTime, DateTime)` |

## CollectionCleansingNode<T>

| Operation | Method |
|---|---|
| Remove nulls | `RemoveNulls()` |
| Remove duplicates | `RemoveDuplicates()` |
| Remove empty | `RemoveEmpty()` |
| Sort | `Sort()` |
| Reverse | `Reverse()` |
| Take | `Take(int)` |
| Skip | `Skip(int)` |

## Error Handling

All validation/cleansing/filtering nodes support per-rule error decisions:

```csharp
cfg.ForProperty(o => o.Email)
    .IsEmail()
    .OnError(ResilienceDecision.Skip); // Skip invalid items

cfg.ForProperty(o => o.Amount)
    .IsPositive()
    .OnError(ResilienceDecision.DeadLetter); // Dead-letter invalid items
```

Default decision: `Fail` (throw exception, stop pipeline unless resilience policy overrides).
