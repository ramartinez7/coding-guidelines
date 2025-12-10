# Extension Methods vs Inheritance

> Extend types without inheritance—extension methods add behavior without coupling, while inheritance creates tight relationships.

## Problem

Inheritance creates tight coupling between base and derived classes. Changes to base classes ripple through hierarchies. Extension methods provide a lightweight alternative for adding behavior to types you don't control or shouldn't modify.

## Example

### ❌ Before (Overuse of Inheritance)

```csharp
// Base class with some utility methods
public class StringUtils
{
    public virtual bool IsValidEmail(string email)
    {
        return email.Contains("@") && email.Contains(".");
    }

    public virtual string Truncate(string text, int maxLength)
    {
        return text.Length <= maxLength ? text : text[..maxLength];
    }
}

// Need email validation? Inherit!
public class UserService : StringUtils
{
    public void RegisterUser(string email)
    {
        if (IsValidEmail(email))
        {
            // ...
        }
    }
}

// Need truncation? Inherit!
public class DisplayFormatter : StringUtils
{
    public string FormatTitle(string title)
    {
        return Truncate(title, 50);
    }
}

// Now both classes are coupled to StringUtils
// Single inheritance means you can't inherit from anything else
// Changes to StringUtils break both classes
```

### ✅ After (Extension Methods)

```csharp
public static class StringExtensions
{
    public static bool IsValidEmail(this string email)
    {
        return email.Contains("@") && email.Contains(".");
    }

    public static string Truncate(this string text, int maxLength)
    {
        return text.Length <= maxLength ? text : text[..maxLength];
    }
}

// Use extension methods directly - no inheritance
public class UserService
{
    public void RegisterUser(string email)
    {
        if (email.IsValidEmail())
        {
            // ...
        }
    }
}

public class DisplayFormatter
{
    public string FormatTitle(string title)
    {
        return title.Truncate(50);
    }
}

// No coupling, no inheritance constraints
// Each class can inherit from whatever it needs
```

## Why Extension Methods Are Better

1. **No coupling**: Classes don't depend on utility base classes.

2. **Composable**: Can use multiple extension method classes.

3. **Extend sealed types**: Can add methods to `string`, `DateTime`, third-party types.

4. **Optional discovery**: Extension methods only appear when namespace is imported.

5. **No inheritance slot**: Preserve inheritance for true "is-a" relationships.

## When to Use Each

### Use Extension Methods

✅ **Adding utilities to types you don't own**:
```csharp
public static class DateTimeExtensions
{
    public static bool IsWeekend(this DateTime date)
    {
        return date.DayOfWeek is DayOfWeek.Saturday or DayOfWeek.Sunday;
    }

    public static DateTime NextBusinessDay(this DateTime date)
    {
        var next = date.AddDays(1);
        while (next.IsWeekend())
            next = next.AddDays(1);
        return next;
    }
}
```

✅ **LINQ-style operations on custom types**:
```csharp
public static class OrderExtensions
{
    public static Money TotalValue(this IEnumerable<Order> orders)
    {
        return orders.Aggregate(Money.Zero, (acc, o) => acc + o.Total);
    }

    public static IEnumerable<Order> PendingOnly(this IEnumerable<Order> orders)
    {
        return orders.Where(o => o.Status == OrderStatus.Pending);
    }
}

// Usage feels natural
var total = orders.PendingOnly().TotalValue();
```

✅ **Convenience methods that don't belong in the type**:
```csharp
public static class ResultExtensions
{
    public static Result<TOut, TError> Map<TIn, TOut, TError>(
        this Result<TIn, TError> result,
        Func<TIn, TOut> mapper)
    {
        return result.IsSuccess
            ? Result<TOut, TError>.Success(mapper(result.Value))
            : Result<TOut, TError>.Failure(result.Error);
    }

    public static Result<T, TError> Flatten<T, TError>(
        this Result<Result<T, TError>, TError> result)
    {
        return result.IsSuccess ? result.Value : Result<T, TError>.Failure(result.Error);
    }
}
```

### Use Inheritance

✅ **True "is-a" relationships**:
```csharp
public abstract class Payment
{
    public Money Amount { get; }
    public abstract PaymentMethod Method { get; }
}

public sealed class CreditCardPayment : Payment
{
    public CardNumber CardNumber { get; }
    public override PaymentMethod Method => PaymentMethod.CreditCard;
}
```

✅ **Polymorphic behavior**:
```csharp
public abstract class ValidationRule
{
    public abstract ValidationResult Validate(Order order);
}

public sealed class MinimumAmountRule : ValidationRule
{
    private readonly Money minimumAmount;

    public override ValidationResult Validate(Order order)
    {
        return order.Total >= minimumAmount
            ? ValidationResult.Valid()
            : ValidationResult.Invalid($"Order must be at least {minimumAmount}");
    }
}
```

✅ **Framework requirements** (ASP.NET controllers, Entity Framework entities):
```csharp
public class OrdersController : ControllerBase
{
    // Must inherit from ControllerBase
}
```

## Extension Method Best Practices

### 1. Make Them Discoverable

Group related extensions in clearly-named static classes:

```csharp
// Good naming
public static class StringValidationExtensions { }
public static class DateTimeExtensions { }
public static class EnumerableExtensions { }

// Poor naming
public static class Helpers { }
public static class Utils { }
public static class Extensions { }
```

### 2. Check for Null

Extension methods can be called on null references:

```csharp
public static class StringExtensions
{
    public static bool IsNullOrEmpty(this string? value)
    {
        return string.IsNullOrEmpty(value);
    }

    public static string Truncate(this string value, int maxLength)
    {
        // Check for null if it shouldn't be null
        if (value is null)
            throw new ArgumentNullException(nameof(value));
        
        return value.Length <= maxLength ? value : value[..maxLength];
    }
}

// Can be called on null
string? text = null;
bool empty = text.IsNullOrEmpty();  // OK - handles null
```

### 3. Don't Overuse

Extension methods should feel natural, not forced:

```csharp
// ✅ Good - natural extension of the type
public static bool IsWeekend(this DateTime date) { }

// ❌ Bad - feels unnatural
public static void SendEmail(this string emailAddress, string subject, string body) { }

// ✅ Better - use a proper service
emailService.Send(new EmailMessage(emailAddress, subject, body));
```

### 4. Avoid Breaking Encapsulation

Don't use extension methods to work around proper encapsulation:

```csharp
public class BankAccount
{
    private decimal balance;  // Private for a reason

    public decimal GetBalance() => balance;
}

// ❌ Bad - trying to mutate private state
public static class BankAccountExtensions
{
    public static void IncreaseBalance(this BankAccount account, decimal amount)
    {
        // Can't access private balance field - good!
        // Don't try to work around this with reflection
    }
}

// ✅ Good - respect encapsulation
public class BankAccount
{
    private decimal balance;

    public void Deposit(Money amount)
    {
        balance += amount.Amount;
    }
}
```

## Combining Extension Methods with Interfaces

Extension methods work well with interfaces for composable behavior:

```csharp
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
}

// Add common operations via extensions
public static class RepositoryExtensions
{
    public static async Task<T> GetByIdOrThrowAsync<T>(
        this IRepository<T> repository,
        int id)
    {
        var entity = await repository.GetByIdAsync(id);
        return entity ?? throw new EntityNotFoundException($"Entity {id} not found");
    }

    public static async Task<bool> ExistsAsync<T>(
        this IRepository<T> repository,
        int id)
    {
        var entity = await repository.GetByIdAsync(id);
        return entity != null;
    }
}

// All IRepository implementations get these methods
var user = await userRepository.GetByIdOrThrowAsync(123);
bool exists = await orderRepository.ExistsAsync(456);
```

## Real-World Example

### Extending Result\<T, E\>

```csharp
public readonly record struct Result<T, TError>
{
    public bool IsSuccess { get; init; }
    public T Value { get; init; }
    public TError Error { get; init; }

    public static Result<T, TError> Success(T value) =>
        new() { IsSuccess = true, Value = value };

    public static Result<T, TError> Failure(TError error) =>
        new() { IsSuccess = false, Error = error };
}

// Add functional operations via extensions
public static class ResultExtensions
{
    public static Result<TOut, TError> Map<TIn, TOut, TError>(
        this Result<TIn, TError> result,
        Func<TIn, TOut> mapper)
    {
        return result.IsSuccess
            ? Result<TOut, TError>.Success(mapper(result.Value))
            : Result<TOut, TError>.Failure(result.Error);
    }

    public static Result<TOut, TError> Bind<TIn, TOut, TError>(
        this Result<TIn, TError> result,
        Func<TIn, Result<TOut, TError>> binder)
    {
        return result.IsSuccess
            ? binder(result.Value)
            : Result<TOut, TError>.Failure(result.Error);
    }

    public static T GetValueOrDefault<T, TError>(
        this Result<T, TError> result,
        T defaultValue)
    {
        return result.IsSuccess ? result.Value : defaultValue;
    }

    public static async Task<Result<TOut, TError>> MapAsync<TIn, TOut, TError>(
        this Result<TIn, TError> result,
        Func<TIn, Task<TOut>> mapper)
    {
        if (!result.IsSuccess)
            return Result<TOut, TError>.Failure(result.Error);

        var value = await mapper(result.Value);
        return Result<TOut, TError>.Success(value);
    }
}

// Usage - fluent, composable
var result = GetUserId()
    .Bind(id => GetUser(id))
    .Map(user => user.Email)
    .GetValueOrDefault(EmailAddress.Default);
```

## See Also

- [SOLID Principles](./solid-principles.md) — composition over inheritance
- [Method Chaining](./method-chaining.md) — fluent APIs with extension methods
- [Specification Pattern](./specification-pattern.md) — composable business rules
- [Type-Safe Builder](./type-safe-builder.md) — building complex objects
