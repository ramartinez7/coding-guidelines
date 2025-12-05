# Nullability vs. Optionality

> Using `null` to represent "no value" when a dedicated `Option<T>` type would express intent more clearly.

## Problem

C#'s nullable reference types (`T?`) conflate multiple meanings: "not yet assigned," "not found," "not applicable," or "intentionally empty." When `null` appears, callers must guess what it means—and the compiler can't enforce handling.

An `Option<T>` (also called `Maybe<T>`) makes optionality explicit and forces callers to acknowledge the absence of a value.

## Example

### ❌ Before

```csharp
public class UserService
{
    // What does null mean here?
    // - User doesn't exist?
    // - Database error?
    // - User is soft-deleted?
    public User? FindByEmail(string email)
    {
        return _db.Users.FirstOrDefault(u => u.Email == email);
    }

    public string GetDisplayName(string email)
    {
        var user = FindByEmail(email);
        return user?.DisplayName ?? "Unknown";  // Null coalescing hides the problem
    }

    public void SendWelcomeEmail(string email)
    {
        var user = FindByEmail(email);
        if (user != null)  // Easy to forget this check
        {
            _emailService.Send(user.Email, "Welcome!");
        }
        // Silently does nothing if user is null—is that intentional?
    }
}

// Nullable chains become hard to follow
var city = user?.Address?.City ?? "Unknown";
```

### ✅ After

```csharp
// A simple Option type
public readonly record struct Option<T> where T : notnull
{
    private readonly T? _value;
    private readonly bool _hasValue;

    private Option(T value) => (_value, _hasValue) = (value, true);

    public static Option<T> Some(T value) => new(value);
    public static Option<T> None => default;

    public bool IsSome => _hasValue;
    public bool IsNone => !_hasValue;

    // Pattern matching
    public TResult Match<TResult>(Func<T, TResult> onSome, Func<TResult> onNone)
        => _hasValue ? onSome(_value!) : onNone();

    public void Match(Action<T> onSome, Action onNone)
    {
        if (_hasValue) onSome(_value!);
        else onNone();
    }

    // Transformations
    
    /// <summary>
    /// Transforms the value if present. Use when your function returns a plain value.
    /// </summary>
    public Option<TResult> Map<TResult>(Func<T, TResult> map) where TResult : notnull
        => _hasValue ? Option<TResult>.Some(map(_value!)) : Option<TResult>.None;

    /// <summary>
    /// Chains optional operations. Use when your function returns an Option.
    /// </summary>
    public Option<TResult> Bind<TResult>(Func<T, Option<TResult>> bind) where TResult : notnull
        => _hasValue ? bind(_value!) : Option<TResult>.None;

    public Option<T> Where(Func<T, bool> predicate)
        => _hasValue && predicate(_value!) ? this : None;

    // Value extraction
    public T GetValueOrDefault(T defaultValue) 
        => _hasValue ? _value! : defaultValue;

    public T GetValueOrDefault(Func<T> defaultFactory) 
        => _hasValue ? _value! : defaultFactory();

    public T GetValueOrThrow(string? message = null)
        => _hasValue ? _value! : throw new InvalidOperationException(message ?? "Option has no value");

    public T GetValueOrThrow(Func<Exception> exceptionFactory)
        => _hasValue ? _value! : throw exceptionFactory();

    public T? GetValueOrNull() where T : class
        => _hasValue ? _value : null;

    // Side effects
    public Option<T> Do(Action<T> action)
    {
        if (_hasValue) action(_value!);
        return this;
    }

    public Option<T> DoWhenNone(Action action)
    {
        if (!_hasValue) action();
        return this;
    }

    // Fallback
    public Option<T> Or(Option<T> fallback)
        => _hasValue ? this : fallback;

    public Option<T> Or(Func<Option<T>> fallbackFactory)
        => _hasValue ? this : fallbackFactory();
}

public class UserService
{
    // Intent is clear: may or may not find a user
    public Option<User> FindByEmail(string email)
    {
        var user = _db.Users.FirstOrDefault(u => u.Email == email);
        return user is not null ? Option<User>.Some(user) : Option<User>.None;
    }

    public string GetDisplayName(string email)
    {
        return FindByEmail(email)
            .Map(user => user.DisplayName)
            .GetValueOrDefault("Unknown");
    }

    public void SendWelcomeEmail(string email)
    {
        FindByEmail(email).Match(
            onSome: user => _emailService.Send(user.Email, "Welcome!"),
            onNone: () => _logger.Warn($"User not found: {email}")
        );
    }
}

// Chaining with Bind is explicit
Option<string> GetUserCity(string email) =>
    FindByEmail(email)
        .Bind(user => user.Address)   // Address is Option<Address>
        .Map(address => address.City);
```

## Understanding Map and Bind

`Map` and `Bind` let you transform the value inside an `Option` without unpacking it first. They handle the "None" case automatically, so you can chain operations without nested `if` checks.

### Map: Transform the Inner Value

`Map` applies a function to the value *if it exists*, wrapping the result in a new `Option`:

```csharp
Option<User> user = FindByEmail(email);

// Without Map: manual unpacking
string displayName;
if (user.IsSome)
    displayName = user.GetValueOrThrow().DisplayName;  // Awkward
else
    displayName = "Unknown";

// With Map: transform inside the Option
Option<string> displayName = user.Map(u => u.DisplayName);
// If user is Some(User), result is Some("John Doe")
// If user is None, result is None (function never called)
```

Think of `Map` like LINQ's `Select`—it transforms what's inside the container.

### Bind: Chain Optional Operations

`Bind` (also called `FlatMap` or `SelectMany`) is for when your transformation *itself* returns an `Option`. It "flattens" the result to avoid `Option<Option<T>>`:

```csharp
// User.Address is Option<Address> (address is optional)
// Address.City is string

Option<User> user = FindByEmail(email);

// Using Map would give Option<Option<Address>> — not what we want
Option<Option<Address>> nested = user.Map(u => u.Address);  // ❌

// Bind flattens it to Option<Address>
Option<Address> address = user.Bind(u => u.Address);  // ✅

// Chain them together
Option<string> city = user
    .Bind(u => u.Address)      // Option<Address>
    .Map(a => a.City);         // Option<string>
```

### The Rule of Thumb

| Your function returns | Use |
|-----------------------|-----|
| `TResult` (plain value) | `Map` |
| `Option<TResult>` | `Bind` |

```csharp
// DisplayName is string → use Map
user.Map(u => u.DisplayName);

// FindManager returns Option<User> → use Bind
user.Bind(u => FindManager(u.DepartmentId));

// Chain them freely
Option<string> managerEmail = user
    .Bind(u => FindManager(u.DepartmentId))  // Option<User>
    .Map(m => m.Email);                       // Option<string>
```

## When to Use `Option<T>` vs. `T?`

| Use `Option<T>` | Use `T?` |
|-----------------|----------|
| Domain concept: "may or may not exist" | Simple DTOs, data transfer |
| You want to force handling at call site | Internal implementation details |
| Chaining optional operations | Performance-critical hot paths |
| Public API boundaries | Interop with existing null-based code |

## Why It's a Problem

- **Ambiguous meaning**: `null` could mean error, not found, or not applicable
- **Silent failures**: Forgetting a null check causes runtime exceptions
- **Hidden control flow**: Null coalescing (`??`) and null propagation (`?.`) hide decision points
- **Viral nullability**: One nullable field forces null checks throughout the codebase
- **No forced handling**: Compiler warns but doesn't prevent ignoring nulls

## Symptoms

- Methods returning `T?` with unclear semantics for `null`
- Long chains of `?.` operators
- Defensive `?? defaultValue` without understanding why
- `if (x != null)` scattered throughout the codebase
- `NullReferenceException` in production logs
- Comments explaining what `null` means: `// null means not found`

## Benefits

- **Explicit intent**: `Option<T>` says "this might not have a value" in the type system
- **Forced handling**: `Match` requires both `Some` and `None` cases
- **Composable**: `Map`, `Bind`, and `Match` chain cleanly
- **Self-documenting**: No need for comments explaining null semantics
- **Compiler-enforced**: Can't forget to handle the empty case

## Practical Considerations

### Converting Between Option and Nullable

```csharp
public static class OptionExtensions
{
    public static Option<T> ToOption<T>(this T? value) where T : class
        => value is not null ? Option<T>.Some(value) : Option<T>.None;

    public static Option<T> ToOption<T>(this T? value) where T : struct
        => value.HasValue ? Option<T>.Some(value.Value) : Option<T>.None;

    public static T? ToNullable<T>(this Option<T> option) where T : class
        => option.Match(v => v, () => null);
}

// At system boundaries
var user = externalApi.GetUser(id).ToOption();  // Convert null to Option
var result = FindUser(id).ToNullable();          // Convert Option to null for DTO
```

### LINQ Integration

```csharp
public static class OptionLinq
{
    public static Option<T> FirstOrNone<T>(this IEnumerable<T> source) where T : notnull
    {
        foreach (var item in source)
            return Option<T>.Some(item);
        return Option<T>.None;
    }

    public static Option<T> FirstOrNone<T>(
        this IEnumerable<T> source, 
        Func<T, bool> predicate) where T : notnull
        => source.Where(predicate).FirstOrNone();
}

// Usage
var user = users.FirstOrNone(u => u.Email == email);
```

## See Also

- [Honest Functions](./honest-functions.md) — Use `Result<T, TError>` when failure has a reason
- [Boolean Blindness](./boolean-blindness.md) — `Option` is clearer than `bool TryGet(out T value)`
- [Ghost States](./ghost-states.md) — `Option` helps model truly optional properties
