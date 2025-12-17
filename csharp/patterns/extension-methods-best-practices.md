# Extension Methods Best Practices

> Use extension methods to add functionality to types you don't own, but avoid overuse and maintain discoverability.

## Problem

Overusing extension methods creates scattered functionality, while proper use enhances readability and maintains separation of concerns.

## Example

### ❌ Before

```csharp
// Utility class with static methods (hard to discover, not fluent)
public static class StringUtils
{
    public static bool IsValidEmail(string email) { }
    public static string ToTitleCase(string input) { }
}

// Usage: not fluent
var isValid = StringUtils.IsValidEmail(email);
var title = StringUtils.ToTitleCase(name);
```

### ✅ After

```csharp
// Extension methods (discoverable, fluent)
public static class StringExtensions
{
    public static bool IsValidEmail(this string email)
    {
        return !string.IsNullOrWhiteSpace(email) && email.Contains("@");
    }

    public static string ToTitleCase(this string input)
    {
        return CultureInfo.CurrentCulture.TextInfo.ToTitleCase(input.ToLower());
    }
}

// Usage: fluent
var isValid = email.IsValidEmail();
var title = name.ToTitleCase();
```

## Best Practices

### 1. Use for Non-Owned Types

```csharp
// ✅ Extending types you don't own
public static class DateTimeExtensions
{
    public static bool IsWeekend(this DateTime date)
    {
        return date.DayOfWeek == DayOfWeek.Saturday ||
               date.DayOfWeek == DayOfWeek.Sunday;
    }

    public static DateTime EndOfDay(this DateTime date)
    {
        return date.Date.AddDays(1).AddTicks(-1);
    }
}

// Usage
if (DateTime.Now.IsWeekend())
{
    // Weekend logic
}
```

### 2. Keep Extensions Focused

```csharp
// ❌ God extension class
public static class Extensions
{
    public static bool IsValid(this string s) { }
    public static int Count(this Order o) { }
    public static void Save(this Customer c) { }
}

// ✅ Focused extension classes
public static class StringValidationExtensions
{
    public static bool IsValidEmail(this string email) { }
    public static bool IsValidUrl(this string url) { }
}

public static class OrderExtensions
{
    public static int GetItemCount(this Order order) { }
    public static decimal GetTotal(this Order order) { }
}
```

### 3. Use for Fluent Interfaces

```csharp
// ✅ Fluent query building
public static class QueryExtensions
{
    public static IQueryable<T> WhereActive<T>(this IQueryable<T> query)
        where T : IActivatable
    {
        return query.Where(x => x.IsActive);
    }

    public static IQueryable<T> OrderByCreatedDate<T>(this IQueryable<T> query)
        where T : IHasCreatedDate
    {
        return query.OrderByDescending(x => x.CreatedAt);
    }
}

// Usage: fluent chaining
var recentActiveOrders = context.Orders
    .WhereActive()
    .OrderByCreatedDate()
    .Take(10);
```

### 4. Don't Replace Instance Methods

```csharp
// ❌ Extension on type you own (use instance method instead)
public class Order
{
    public List<OrderItem> Items { get; }
}

public static class OrderExtensions
{
    public static decimal GetTotal(this Order order)  // Should be instance method!
    {
        return order.Items.Sum(i => i.Price);
    }
}

// ✅ Use instance method on owned types
public class Order
{
    public List<OrderItem> Items { get; }

    public decimal GetTotal()
    {
        return Items.Sum(i => i.Price);
    }
}
```

### 5. Use Null Checking

```csharp
// ✅ Always check for null
public static class StringExtensions
{
    public static bool IsNullOrEmpty(this string? value)
    {
        return string.IsNullOrEmpty(value);
    }

    public static string Truncate(this string value, int maxLength)
    {
        ArgumentNullException.ThrowIfNull(value);

        return value.Length <= maxLength
            ? value
            : value[..maxLength];
    }
}
```

### 6. Use for LINQ-Style Operations

```csharp
// ✅ LINQ-style collection extensions
public static class EnumerableExtensions
{
    public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T?> source)
        where T : class
    {
        return source.Where(x => x != null)!;
    }

    public static Dictionary<TKey, TValue> ToDictionarySafe<TKey, TValue>(
        this IEnumerable<TValue> source,
        Func<TValue, TKey> keySelector)
        where TKey : notnull
    {
        var result = new Dictionary<TKey, TValue>();
        foreach (var item in source)
        {
            var key = keySelector(item);
            result[key] = item;  // Overwrites duplicates instead of throwing
        }
        return result;
    }
}
```

### 7. Provide Overloads

```csharp
// ✅ Provide convenient overloads
public static class ResultExtensions
{
    public static Result<TOut, TError> Map<TIn, TOut, TError>(
        this Result<TIn, TError> result,
        Func<TIn, TOut> mapper)
    {
        return result.IsSuccess
            ? Result.Success<TOut, TError>(mapper(result.Value))
            : Result.Failure<TOut, TError>(result.Error);
    }

    public static async Task<Result<TOut, TError>> MapAsync<TIn, TOut, TError>(
        this Result<TIn, TError> result,
        Func<TIn, Task<TOut>> mapper)
    {
        return result.IsSuccess
            ? Result.Success<TOut, TError>(await mapper(result.Value))
            : Result.Failure<TOut, TError>(result.Error);
    }
}
```

### 8. Use Descriptive Names

```csharp
// ❌ Generic names
public static class Extensions
{
    public static string Process(this string s) { }  // What does this do?
    public static T Do<T>(this T item) { }
}

// ✅ Clear, specific names
public static class StringExtensions
{
    public static string RemoveWhitespace(this string value) { }
    public static string ToKebabCase(this string value) { }
}
```

### 9. Document Complex Extensions

```csharp
/// <summary>
/// Retries an async operation with exponential backoff.
/// </summary>
/// <param name="operation">The async operation to retry</param>
/// <param name="maxAttempts">Maximum number of retry attempts</param>
/// <param name="initialDelay">Initial delay before first retry</param>
/// <returns>Result of the operation or the last exception</returns>
public static async Task<T> RetryAsync<T>(
    this Func<Task<T>> operation,
    int maxAttempts = 3,
    TimeSpan? initialDelay = null)
{
    var delay = initialDelay ?? TimeSpan.FromSeconds(1);
    Exception? lastException = null;

    for (int attempt = 0; attempt < maxAttempts; attempt++)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex)
        {
            lastException = ex;
            if (attempt < maxAttempts - 1)
            {
                await Task.Delay(delay);
                delay = TimeSpan.FromMilliseconds(delay.TotalMilliseconds * 2);
            }
        }
    }

    throw lastException!;
}
```

### 10. Avoid State in Extension Methods

```csharp
// ❌ Stateful extension (don't do this!)
public static class BadExtensions
{
    private static int callCount = 0;  // Shared state!

    public static void ProcessWithCount(this Order order)
    {
        callCount++;  // Not thread-safe, confusing
        // Process
    }
}

// ✅ Stateless extensions
public static class GoodExtensions
{
    public static OrderSummary ToSummary(this Order order)
    {
        return new OrderSummary
        {
            Id = order.Id,
            Total = order.Total,
            ItemCount = order.Items.Count
        };
    }
}
```

### 11. Use for Type Conversions

```csharp
// ✅ Conversion extensions
public static class ConversionExtensions
{
    public static Option<int> ToInt(this string value)
    {
        return int.TryParse(value, out var result)
            ? Option.Some(result)
            : Option.None<int>();
    }

    public static DateTime ToUtc(this DateTime dateTime)
    {
        return dateTime.Kind == DateTimeKind.Utc
            ? dateTime
            : DateTime.SpecifyKind(dateTime, DateTimeKind.Utc);
    }
}
```

### 12. Namespace Carefully

```csharp
// ✅ Put extensions in namespace that makes sense
namespace ECommerce.Orders.Extensions  // Order-specific extensions
{
    public static class OrderExtensions
    {
        public static bool IsExpired(this Order order) { }
    }
}

namespace ECommerce.Infrastructure.Extensions  // General infrastructure
{
    public static class DateTimeExtensions
    {
        public static bool IsWeekend(this DateTime date) { }
    }
}
```

## Symptoms

- Difficulty discovering functionality
- Confusion about where methods are defined
- Namespace pollution with too many extensions
- Extension methods on types you own
- Stateful or complex extension logic

## Benefits

- **Fluent API** with method chaining
- **Discoverability** through IntelliSense
- **Extending** types you don't own
- **Cleaner code** with domain-specific operations
- **Separation of concerns** from core types

## See Also

- [Extension Methods vs Inheritance](./extension-methods-vs-inheritance.md) — When to use each
- [Method Chaining](./method-chaining.md) — Fluent interfaces
- [LINQ Query Optimization](./linq-query-optimization.md) — LINQ extensions
