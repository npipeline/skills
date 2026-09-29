# Validation and Cleansing Reference

All rules and operations take a property selector as their first argument. There is no `ForProperty(...)` wrapper. Multiple properties are validated by chaining methods, each with its own selector.

## StringValidationNode<T>

| Rule | Method | Description |
|---|---|---|
| Not empty | `IsNotEmpty(selector, errorMessage?)` | Fails if null or "" |
| Not whitespace | `IsNotWhitespace(selector, errorMessage?)` | Fails if null, "", or whitespace-only |
| Min length | `HasMinLength(selector, int, errorMessage?)` | Minimum character count |
| Max length | `HasMaxLength(selector, int, errorMessage?)` | Maximum character count |
| Length range | `HasLengthBetween(selector, int, int, errorMessage?)` | Between two lengths |
| Email | `IsEmail(selector, errorMessage?)` | Email format |
| URL | `IsUrl(selector, errorMessage?)` | URL format |
| GUID | `IsGuid(selector, errorMessage?)` | GUID format |
| Alphabetic | `IsAlphabetic(selector, errorMessage?)` | Only letters |
| Alphanumeric | `IsAlphanumeric(selector, errorMessage?)` | Only letters and digits |
| Digits only | `IsDigitsOnly(selector, errorMessage?)` | Only digits |
| Numeric | `IsNumeric(selector, errorMessage?)` | Parses as a number |
| Regex | `Matches(selector, string pattern, errorMessage?)` | Custom regex |
| Prefix / suffix | `StartsWith(...)`, `EndsWith(...)` | Prefix or suffix |
| Contains | `Contains(selector, string, errorMessage?)` | Substring present |
| In list | `IsInList(selector, IEnumerable<string>, errorMessage?)` | Member of a set |

### Example

```csharp
builder.AddStringValidation<Order>(cfg => cfg
    .IsNotEmpty(o => o.Email)
    .IsEmail(o => o.Email)
    .HasMaxLength(o => o.Email, 255));
```

## NumericValidationNode<T>

Supports `int`, `double`, `decimal`, and their nullable variants (methods are generic over the numeric type).

| Rule | Method |
|---|---|
| Positive | `IsPositive(selector, ...)` |
| Negative | `IsNegative(selector, ...)` |
| Non-zero | `IsNonZero(selector, ...)` |
| Zero or positive | `IsZeroOrPositive(selector, ...)` |
| Not negative | `IsNotNegative(selector, ...)` |
| Greater than | `IsGreaterThan(selector, T, ...)` |
| Less than | `IsLessThan(selector, T, ...)` |
| Between | `IsBetween(selector, T min, T max, ...)` |
| Integer value | `IsIntegerValue(selector, ...)` |
| Even / odd | `IsEven(selector, ...)`, `IsOdd(selector, ...)` |
| Finite | `IsFinite(selector, ...)` |
| Not null | `IsNotNull(selector, ...)` |

### Example

```csharp
builder.AddNumericValidation<Order>(cfg => cfg
    .IsPositive(o => o.Amount)
    .IsLessThan(o => o.Amount, 100_000)
    .IsGreaterThan(o => o.Quantity, 0)
    .IsIntegerValue(o => o.Quantity));
```

## DateTimeValidationNode<T>

| Rule | Method |
|---|---|
| In future | `IsInFuture(selector, ...)` |
| In past | `IsInPast(selector, ...)` |
| After / before | `IsAfter(...)`, `IsBefore(...)` |
| Between | `IsBetween(selector, DateTime min, DateTime max, ...)` |
| UTC | `IsUtc(selector, ...)` |
| Local | `IsLocal(selector, ...)` |
| Weekday / weekend | `IsWeekday(selector, ...)`, `IsWeekend(selector, ...)` |
| Day of week | `IsDayOfWeek(selector, DayOfWeek, ...)` |
| Today | `IsToday(selector, ...)` |
| In year / month | `IsInYear(...)`, `IsInMonth(...)` |
| Not null | `IsNotNull(selector, ...)` |
| Not min/max | `IsNotMinValue(...)`, `IsNotMaxValue(...)` |

## CollectionValidationNode<T>

| Rule | Method |
|---|---|
| Not empty | `IsNotEmpty(selector, ...)` |
| Min / max count | `HasMinCount(selector, int, ...)`, `HasMaxCount(selector, int, ...)` |
| Count range | `HasCountBetween(selector, int min, int max, ...)` |
| All match | `AllMatch(selector, Func<TItem,bool>, ...)` |
| Any match | `AnyMatch(selector, Func<TItem,bool>, ...)` |
| None match | `NoneMatch(selector, Func<TItem,bool>, ...)` |
| All unique | `AllUnique(selector, ...)` |
| Contains | `Contains(selector, TItem, ...)` / `DoesNotContain(...)` |
| Subset | `IsSubsetOf(selector, IEnumerable<TItem>, ...)` |

## StringCleansingNode<T>

| Operation | Method |
|---|---|
| Trim | `Trim(selector)` |
| Trim start / end | `TrimStart(selector)`, `TrimEnd(selector)` |
| Collapse whitespace | `CollapseWhitespace(selector)` |
| Remove whitespace | `RemoveWhitespace(selector)` |
| To upper / lower / title | `ToUpper(selector)`, `ToLower(selector)`, `ToTitleCase(selector)` |
| Remove special chars | `RemoveSpecialCharacters(selector)` |
| Remove digits | `RemoveDigits(selector)` |
| Remove non-ASCII | `RemoveNonAscii(selector)` |
| Truncate | `Truncate(selector, int maxLength)` |
| Ensure prefix / suffix | `EnsurePrefix(selector, string)`, `EnsureSuffix(selector, string)` |
| Replace | `Replace(selector, string old, string new)` |
| Default if null/whitespace | `DefaultIfNullOrWhitespace(selector, string)` |
| Default if null/empty | `DefaultIfNullOrEmpty(selector, string)` |
| Null if whitespace | `NullIfWhitespace(selector)` |

## NumericCleansingNode<T>

| Operation | Method |
|---|---|
| Clamp | `Clamp(selector, T min, T max)` |
| Min / max | `Min(selector, T)`, `Max(selector, T)` |
| Round | `Round(selector, int decimals)` |
| Floor / ceiling | `Floor(selector)`, `Ceiling(selector)` |
| Absolute value | `AbsoluteValue(selector)` |
| Scale | `Scale(selector, double factor)` |
| Default if null | `DefaultIfNull(selector, T)` |
| To zero if negative | `ToZeroIfNegative(selector)` |

## DateTimeCleansingNode<T>

| Operation | Method |
|---|---|
| Specify kind | `SpecifyKind(selector, DateTimeKind)` |
| To UTC / local | `ToUtc(selector)`, `ToLocal(selector)` |
| Strip time | `StripTime(selector)` |
| Truncate | `Truncate(selector, TimeSpan)` |
| Round to minute/hour/day | `RoundToMinute(selector)`, `RoundToHour(selector)`, `RoundToDay(selector)` |
| Clamp | `Clamp(selector, DateTime min, DateTime max)` |
| Default if null/min/max | `DefaultIfNull(...)`, `DefaultIfMinValue(...)`, `DefaultIfMaxValue(...)` |

## CollectionCleansingNode<T>

| Operation | Method |
|---|---|
| Remove nulls | `RemoveNulls(selector)` |
| Remove duplicates | `RemoveDuplicates(selector, comparer?)` |
| Remove empty | `RemoveEmpty(selector)` |
| Remove whitespace | `RemoveWhitespace(selector)` |
| Sort | `Sort(selector, comparer?)` |
| Reverse | `Reverse(selector)` |
| Take / skip | `Take(selector, int)`, `Skip(selector, int)` |
