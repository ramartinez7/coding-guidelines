# Option Monad (Explicit Optionality)

> Using null to represent "no value" or optional properties—use Option types to make optionality explicit, composable, and null-safe.

## Problem

Nullable references (`string?`) indicate potential absence but don't provide operations for working with optional values. This leads to scattered null checks, `NullReferenceException` risks, and unclear intent about whether null is a valid state or an error.

## Example

### ❌ Before

```csharp
public class UserService
{
    // Is null an error or just "not found"? Type doesn't say.
    public User? GetUser(int id)
    {
        var user = _repository.FindById(id);
        return user; // null if not found—caller must remember to check
    }
    
    // Null checks scattered everywhere
    public string GetUserDisplayName(int userId)
    {
        var user = GetUser(userId);
        
        if (user == null)
            return "Unknown User";
        
        if (user.PreferredName != null)
            return user.PreferredName;
        
        if (user.FirstName != null && user.LastName != null)
            return $"{user.FirstName} {user.LastName}";
        
        if (user.Email != null)
            return user.Email;
        
        return "Anonymous";
    }
    
    // Easy to forget null check
    public void ProcessUser(int userId)
    {
        var user = GetUser(userId);
        SendEmail(user.Email);  // 💥 NullReferenceException if user is null
    }
}
```

**Problems:**
- Null checks scattered throughout code
- Easy to forget null checks
- Unclear whether null is valid or error
- No composable operations on optional values
- `NullReferenceException` at runtime

### ✅ After (Option Monad)

```csharp
/// <summary>
/// Option represents a value that may or may not exist.
/// Makes optionality explicit and composable.
/// </summary>
public abstract record Option<T>
{
    private Option() { }
    
    public sealed record Some(T Value) : Option<T>;
    public sealed record None : Option<T>;
    
    public bool IsSome => this is Some;
    public bool IsNone => this is None;
    
    // Factory methods
    public static Option<T> FromValue(T value) =>
        value is null ? new None() : new Some(value);
    
    public static Option<T> FromNullable(T? value) where T : class =>
        value is not null ? new Some(value) : new None();
}

// Extension methods for Option
public static class OptionExtensions
{
    /// <summary>
    /// Pattern match on option—forces handling both cases.
    /// </summary>
    public static TResult Match<T, TResult>(
        this Option<T> option,
        Func<T, TResult> onSome,
        Func<TResult> onNone)
    {
        return option switch
        {
            Option<T>.Some some => onSome(some.Value),
            Option<T>.None => onNone(),
            _ => throw new InvalidOperationException()
        };
    }
    
    /// <summary>
    /// Transform the value if present—None passes through.
    /// </summary>
    public static Option<TResult> Map<T, TResult>(
        this Option<T> option,
        Func<T, TResult> mapper)
    {
        return option switch
        {
            Option<T>.Some some => Option<TResult>.FromValue(mapper(some.Value)),
            Option<T>.None => Option<TResult>.None,
            _ => throw new InvalidOperationException()
        };
    }
    
    /// <summary>
    /// Chain operations that return Option—stops on first None.
    /// </summary>
    public static Option<TResult> Bind<T, TResult>(
        this Option<T> option,
        Func<T, Option<TResult>> binder)
    {
        return option switch
        {
            Option<T>.Some some => binder(some.Value),
            Option<T>.None => Option<TResult>.None,
            _ => throw new InvalidOperationException()
        };
    }
    
    /// <summary>
    /// Filter option based on predicate.
    /// </summary>
    public static Option<T> Filter<T>(
        this Option<T> option,
        Func<T, bool> predicate)
    {
        return option switch
        {
            Option<T>.Some some when predicate(some.Value) => option,
            _ => Option<T>.None
        };
    }
    
    /// <summary>
    /// Get value or default.
    /// </summary>
    public static T GetValueOrDefault<T>(
        this Option<T> option,
        T defaultValue)
    {
        return option.Match(
            onSome: value => value,
            onNone: () => defaultValue);
    }
    
    /// <summary>
    /// Get value or compute default.
    /// </summary>
    public static T GetValueOrElse<T>(
        this Option<T> option,
        Func<T> getDefault)
    {
        return option.Match(
            onSome: value => value,
            onNone: getDefault);
    }
}

public class UserService
{
    // Type signature is explicit: user might not exist
    public Option<User> GetUser(UserId id)
    {
        var user = _repository.FindById(id);
        return Option<User>.FromNullable(user);
    }
    
    // Composable operations—no scattered null checks
    public string GetUserDisplayName(UserId userId)
    {
        return GetUser(userId)
            .Map(user => user.PreferredName)
            .Filter(name => !string.IsNullOrWhiteSpace(name))
            .Match(
                onSome: name => name,
                onNone: () => GetUserFullName(userId)
                    .GetValueOrElse(() => GetUserEmail(userId)
                        .GetValueOrDefault("Anonymous")));
    }
    
    Option<string> GetUserFullName(UserId userId)
    {
        return GetUser(userId)
            .Bind(user =>
            {
                if (user.FirstName is not null && user.LastName is not null)
                    return Option<string>.FromValue($"{user.FirstName} {user.LastName}");
                
                return new Option<string>.None();
            });
    }
    
    Option<string> GetUserEmail(UserId userId)
    {
        return GetUser(userId)
            .Map(user => user.Email)
            .Filter(email => !string.IsNullOrWhiteSpace(email));
    }
    
    // Cannot forget to check—forced by type
    public Result<Unit, Error> ProcessUser(UserId userId)
    {
        return GetUser(userId).Match(
            onSome: user =>
            {
                SendEmail(user.Email);
                return Result<Unit, Error>.Ok(Unit.Value);
            },
            onNone: () =>
                Result<Unit, Error>.Fail(new Error.UserNotFound(userId)));
    }
}
```

## Option with Records

```csharp
public sealed record Product(
    ProductId Id,
    ProductName Name,
    Money Price,
    Option<string> Description,  // Explicitly optional
    Option<Discount> Discount);  // Explicitly optional

// Create product with optional fields
var product = new Product(
    ProductId.NewId(),
    ProductName.From("Widget"),
    Money.USD(19.99m),
    Option<string>.FromValue("A great widget"),
    Option<Discount>.None);

// Pattern match on optional fields
var description = product.Description.Match(
    onSome: desc => desc,
    onNone: () => "No description available");

var finalPrice = product.Discount.Match(
    onSome: discount => product.Price * (1 - discount.Percentage),
    onNone: () => product.Price);
```

## Option in Domain Models

```csharp
public sealed record Order(
    OrderId Id,
    CustomerId CustomerId,
    IReadOnlyList<OrderItem> Items,
    OrderStatus Status,
    Option<ShippingInfo> ShippingInfo)  // Only present when shipped
{
    // Type-safe access to optional shipping info
    public Option<TrackingNumber> GetTrackingNumber() =>
        ShippingInfo.Map(info => info.TrackingNumber);
    
    public Option<DateTime> GetEstimatedDelivery() =>
        ShippingInfo.Map(info => info.EstimatedDelivery);
}

public sealed record ShippingInfo(
    DateTime ShippedAt,
    TrackingNumber TrackingNumber,
    DateTime EstimatedDelivery,
    ShippingAddress Address);

// Usage
var order = GetOrder(orderId);

var trackingMessage = order.GetTrackingNumber().Match(
    onSome: tracking => $"Tracking number: {tracking}",
    onNone: () => "Not yet shipped");

var deliveryMessage = order.GetEstimatedDelivery().Match(
    onSome: date => $"Estimated delivery: {date:d}",
    onNone: () => "Delivery date not available");
```

## Async Option Operations

```csharp
public static class AsyncOptionExtensions
{
    public static async Task<Option<TResult>> MapAsync<T, TResult>(
        this Task<Option<T>> optionTask,
        Func<T, TResult> mapper)
    {
        var option = await optionTask;
        return option.Map(mapper);
    }
    
    public static async Task<Option<TResult>> BindAsync<T, TResult>(
        this Task<Option<T>> optionTask,
        Func<T, Task<Option<TResult>>> binder)
    {
        var option = await optionTask;
        
        return option switch
        {
            Option<T>.Some some => await binder(some.Value),
            Option<T>.None => new Option<TResult>.None(),
            _ => throw new InvalidOperationException()
        };
    }
    
    public static async Task<TResult> MatchAsync<T, TResult>(
        this Task<Option<T>> optionTask,
        Func<T, Task<TResult>> onSome,
        Func<Task<TResult>> onNone)
    {
        var option = await optionTask;
        
        return option switch
        {
            Option<T>.Some some => await onSome(some.Value),
            Option<T>.None => await onNone(),
            _ => throw new InvalidOperationException()
        };
    }
}

// Async chain
public async Task<Option<OrderSummary>> GetOrderSummaryAsync(OrderId orderId)
{
    return await GetOrderAsync(orderId)
        .BindAsync(order => GetCustomerAsync(order.CustomerId))
        .MapAsync(customer => new OrderSummary(orderId, customer.Name));
}
```

## Combining Options

```csharp
/// <summary>
/// Combine two options—Some only if both are Some.
/// </summary>
public static Option<(T1, T2)> Zip<T1, T2>(
    this Option<T1> option1,
    Option<T2> option2)
{
    return option1.Bind(v1 =>
        option2.Map(v2 => (v1, v2)));
}

/// <summary>
/// Apply function in option to value in option.
/// </summary>
public static Option<TResult> Apply<T, TResult>(
    this Option<Func<T, TResult>> optionFunc,
    Option<T> optionValue)
{
    return optionFunc.Bind(func =>
        optionValue.Map(func));
}

// Usage
var firstName = Option<string>.FromValue("John");
var lastName = Option<string>.FromValue("Doe");

var fullName = firstName.Zip(lastName)
    .Map(names => $"{names.Item1} {names.Item2}");
// fullName is Some("John Doe")

var firstName2 = new Option<string>.None();
var fullName2 = firstName2.Zip(lastName);
// fullName2 is None
```

## Option vs Nullable

```csharp
/// <summary>
/// Convert nullable reference to Option.
/// </summary>
public static Option<T> ToOption<T>(this T? value) where T : class
{
    return Option<T>.FromNullable(value);
}

/// <summary>
/// Convert nullable struct to Option.
/// </summary>
public static Option<T> ToOption<T>(this T? value) where T : struct
{
    return value.HasValue
        ? Option<T>.FromValue(value.Value)
        : new Option<T>.None();
}

/// <summary>
/// Convert Option to nullable reference.
/// </summary>
public static T? ToNullable<T>(this Option<T> option) where T : class
{
    return option.Match(
        onSome: value => value,
        onNone: () => null);
}

// Usage
string? nullableString = GetNullableString();
Option<string> option = nullableString.ToOption();

int? nullableInt = GetNullableInt();
Option<int> optionInt = nullableInt.ToOption();
```

## Option in Collections

```csharp
/// <summary>
/// Get first element as Option—None if empty.
/// </summary>
public static Option<T> FirstOrNone<T>(this IEnumerable<T> source)
{
    foreach (var item in source)
        return Option<T>.FromValue(item);
    
    return new Option<T>.None();
}

/// <summary>
/// Get first element matching predicate as Option.
/// </summary>
public static Option<T> FirstOrNone<T>(
    this IEnumerable<T> source,
    Func<T, bool> predicate)
{
    foreach (var item in source)
    {
        if (predicate(item))
            return Option<T>.FromValue(item);
    }
    
    return new Option<T>.None();
}

/// <summary>
/// Filter out None values from collection of Options.
/// </summary>
public static IEnumerable<T> WhereSome<T>(this IEnumerable<Option<T>> source)
{
    foreach (var option in source)
    {
        if (option is Option<T>.Some some)
            yield return some.Value;
    }
}

// Usage
var users = new[] { user1, user2, user3 };

var firstAdmin = users
    .FirstOrNone(u => u.IsAdmin)
    .Match(
        onSome: admin => $"Admin: {admin.Name}",
        onNone: () => "No admin found");

var emailAddresses = users
    .Select(u => u.Email.ToOption())
    .WhereSome()
    .ToList();
```

## Option with Result

```csharp
/// <summary>
/// Convert Option to Result—specify error when None.
/// </summary>
public static Result<T, TError> ToResult<T, TError>(
    this Option<T> option,
    TError error)
{
    return option.Match(
        onSome: value => Result<T, TError>.Ok(value),
        onNone: () => Result<T, TError>.Fail(error));
}

/// <summary>
/// Convert Result to Option—discard error information.
/// </summary>
public static Option<T> ToOption<T, TError>(this Result<T, TError> result)
{
    return result.Match(
        onSuccess: value => Option<T>.FromValue(value),
        onFailure: _ => new Option<T>.None());
}

// Usage
var userOption = GetUser(userId);

var userResult = userOption.ToResult(
    new Error.UserNotFound(userId));

return userResult.Match(
    onSuccess: user => ProcessUser(user),
    onFailure: error => HandleError(error));
```

## LINQ Query Syntax Support

```csharp
public static class OptionLinqExtensions
{
    // Enable LINQ query syntax for Option
    public static Option<TResult> Select<T, TResult>(
        this Option<T> option,
        Func<T, TResult> selector)
    {
        return option.Map(selector);
    }
    
    public static Option<TResult> SelectMany<T, TResult>(
        this Option<T> option,
        Func<T, Option<TResult>> selector)
    {
        return option.Bind(selector);
    }
    
    public static Option<TResult> SelectMany<T, TIntermediate, TResult>(
        this Option<T> option,
        Func<T, Option<TIntermediate>> selector,
        Func<T, TIntermediate, TResult> resultSelector)
    {
        return option.Bind(t =>
            selector(t).Map(i => resultSelector(t, i)));
    }
    
    public static Option<T> Where<T>(
        this Option<T> option,
        Func<T, bool> predicate)
    {
        return option.Filter(predicate);
    }
}

// LINQ query syntax
var result = from user in GetUser(userId)
             from address in user.ShippingAddress.ToOption()
             where address.IsValid
             select address.ToString();
```

## Key Principles

### 1. Explicit Optionality

```csharp
// ❌ Unclear if null is valid
public User? GetUser(int id);

// ✅ Explicitly optional
public Option<User> GetUser(UserId id);
```

### 2. Composable Operations

```csharp
// Chain operations without null checks
return GetUser(id)
    .Map(user => user.Email)
    .Filter(email => IsValid(email))
    .GetValueOrDefault("no-reply@example.com");
```

### 3. Forced Handling

```csharp
// Must handle both Some and None cases
option.Match(
    onSome: value => { /* handle value */ },
    onNone: () => { /* handle absence */ });
```

## Why It's a Problem

1. **NullReferenceException**: Forgot to check for null
2. **Scattered null checks**: Same checks repeated everywhere
3. **Lost intent**: Is null an error or valid state?
4. **Not composable**: Cannot chain operations on nullable values
5. **Runtime failures**: Null checks missed until production

## Symptoms

- Frequent `NullReferenceException` in production
- Null checks scattered throughout code
- Defensive coding with redundant null checks
- Comments like "// can be null in some cases"
- Unclear API contracts about optionality

## Benefits

- **Explicit optionality**: Type shows value might not exist
- **Composable**: Chain operations cleanly
- **No NullReferenceException**: Forced to handle absence
- **Self-documenting**: Option in signature shows intent
- **Type-safe**: Compiler ensures handling

## Trade-offs

- **Learning curve**: Monadic operations unfamiliar
- **Verbosity**: More code than nullable types
- **Library integration**: Some libraries expect nullable
- **Pattern matching**: Requires understanding of functional concepts

## See Also

- [Result Monad](./result-monad.md) — for operations that can fail with errors
- [Nullability vs. Optionality](./nullability-optionality.md) — when to use each
- [Honest Functions](./honest-functions.md) — explicit outcomes in signatures
- [Smart Constructors](./smart-constructors.md) — parse, don't validate
