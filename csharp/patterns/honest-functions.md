# Honest Functions (The Result Pattern)

> Functions that lie about their return type—use Result types to make failure explicit in the signature.

## Problem

A function signature should tell the whole truth. If `GetUser` returns `User` but might throw or return `null`, it's lying. Callers must read documentation, check source code, or discover failures at runtime.

## Example

### ❌ Before

```csharp
public class UserRepository
{
    // Signature says: "Give me an ID, I give you a User."
    // Reality: "I might give you a User, return null, or blow up."
    public User GetUser(int id)
    {
        if (id <= 0) throw new ArgumentException("Id must be positive");
        
        var user = _db.Find(id);
        if (user == null) return null;  // What does null mean here?
        
        return user;
    }

    public void UpdateEmail(int id, string email)
    {
        var user = GetUser(id);
        user.Email = email;  // NullReferenceException waiting to happen
    }
}

// Caller must remember to check null AND wrap in try-catch
try
{
    var user = repo.GetUser(id);
    if (user != null)
    {
        // finally safe to use
    }
}
catch (ArgumentException) { /* handle */ }
```

### ✅ After

```csharp
// A simple Result type
public readonly record struct Result<TValue, TError>
{
    public TValue? Value { get; }
    public TError? Error { get; }
    public bool IsSuccess { get; }

    private Result(TValue value) => (Value, IsSuccess) = (value, true);
    private Result(TError error) => (Error, IsSuccess) = (error, false);

    public static Result<TValue, TError> Success(TValue value) => new(value);
    public static Result<TValue, TError> Failure(TError error) => new(error);

    public TResult Match<TResult>(Func<TValue, TResult> onSuccess, Func<TError, TResult> onFailure)
        => IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}

// Error types that describe what went wrong
public abstract record GetUserError;
public sealed record UserNotFound(int Id) : GetUserError;
public sealed record InvalidUserId(int Id, string Reason) : GetUserError;

public class UserRepository
{
    // Signature tells the truth: "I return a User OR an error explaining why I couldn't"
    public Result<User, GetUserError> GetUser(int id)
    {
        if (id <= 0)
            return Result<User, GetUserError>.Failure(new InvalidUserId(id, "Id must be positive"));
        
        var user = _db.Find(id);
        if (user == null)
            return Result<User, GetUserError>.Failure(new UserNotFound(id));
        
        return Result<User, GetUserError>.Success(user);
    }

    public Result<User, GetUserError> UpdateEmail(int id, string email)
    {
        return GetUser(id).Match(
            onSuccess: user =>
            {
                user.Email = email;
                return Result<User, GetUserError>.Success(user);
            },
            onFailure: error => Result<User, GetUserError>.Failure(error)
        );
    }
}

// Caller is forced to handle both cases
var result = repo.GetUser(42);
result.Match(
    onSuccess: user => Console.WriteLine($"Found: {user.Name}"),
    onFailure: error => error switch
    {
        UserNotFound e => Console.WriteLine($"No user with ID {e.Id}"),
        InvalidUserId e => Console.WriteLine($"Bad ID {e.Id}: {e.Reason}"),
        _ => Console.WriteLine("Unknown error")
    }
);
```

## Why It's a Problem

1. **Dishonest signature**: `User GetUser(int id)` implies success is guaranteed—it's not.

2. **Exceptions as flow control**: Using exceptions for expected scenarios (user not found) is expensive and hides logic flow.

3. **Null ambiguity**: Does `null` mean "database error", "user not found", or "not initialized"? Impossible to tell.

4. **Forgotten checks**: Nothing forces callers to handle the failure case—bugs appear at runtime.

## Symptoms

- `try-catch` blocks used for business logic (e.g., `catch (UserNotFoundException)`)
- Checking for `null` immediately after calling a method
- Comments saying "Returns null if not found" or "Throws if invalid"
- Methods with `OrDefault` or `OrNull` suffixes
- Defensive `if (x != null)` scattered throughout the codebase

## Benefits

- **Compile-time error handling**: Can't access the value without handling the error case
- **Self-documenting**: `Result<User, GetUserError>` tells callers exactly what can happen
- **Explicit error types**: `UserNotFound` vs `InvalidUserId` vs `DatabaseError`—no ambiguity
- **Composable**: Results can be chained with `Match`, `Map`, `Bind` operations
- **No exceptions for expected failures**: Exceptions reserved for truly exceptional circumstances

## Variations

### Simple Result (No Error Details)

```csharp
public readonly record struct Option<T>
{
    public T? Value { get; }
    public bool HasValue { get; }

    private Option(T value) => (Value, HasValue) = (value, true);
    
    public static Option<T> Some(T value) => new(value);
    public static Option<T> None => default;
}

public Option<User> FindUser(int id)
{
    var user = _db.Find(id);
    return user != null ? Option<User>.Some(user) : Option<User>.None;
}
```

### Using OneOf Library

```csharp
// Using the OneOf NuGet package
public OneOf<User, NotFound, ValidationError> GetUser(int id)
{
    if (id <= 0) return new ValidationError("Id must be positive");
    var user = _db.Find(id);
    return user ?? (OneOf<User, NotFound, ValidationError>)new NotFound();
}
```

### Nullable Reference Types (C# 8+)

For simple "found or not found" scenarios:

```csharp
public User? FindUser(int id)  // Nullable return type is honest
{
    return _db.Find(id);  // Caller knows to expect null
}
```

## When to Use What

| Scenario | Approach |
|----------|----------|
| Simple "found or not" | `T?` (nullable) or `Option<T>` |
| Need error details | `Result<T, TError>` |
| Multiple error types | `OneOf<T, Error1, Error2>` or error hierarchy |
| Truly exceptional failures | Exceptions (database down, out of memory) |

## See Also

- [Primitive Obsession](./primitive-obsession.md) — wrapping primitives in types
- [Static Factory Methods](./static-factory-methods.md) — construction with validation
- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md) — modeling variants
- [Typed Errors](./typed-errors.md) — making failure cases explicit with types
- [Smart Constructors](./smart-constructors.md) — parse, don't validate
