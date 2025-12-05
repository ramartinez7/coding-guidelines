# Boolean Blindness

> Methods returning `bool` that force callers to remember what `true` and `false` mean—use descriptive types instead.

## Problem

When a method returns `bool`, the meaning of `true` and `false` is invisible at the call site. Callers must remember (or look up) what the boolean represents, leading to inverted logic bugs and unreadable code.

## Example

### ❌ Before

```csharp
public class UserService
{
    public bool ValidateUser(string username, string password)
    {
        // Returns true if valid, false if invalid... or is it the other way?
        return _db.CheckCredentials(username, password);
    }

    public bool IsLocked(int userId)
    {
        return _db.GetLockStatus(userId);
    }
}

// Caller - what does 'true' mean here?
if (userService.ValidateUser(username, password))
{
    // Is this the success path or failure path?
    // Have to check the method to know
}

// Easy to invert by mistake
if (!userService.IsLocked(userId))  // Wait, did I get this backwards?
{
    AllowAccess();
}

// Boolean parameters are even worse
void ProcessOrder(int orderId, bool sendEmail, bool priority, bool validateStock)
{
    // ...
}

// What do these booleans mean?!
ProcessOrder(123, true, false, true);
```

### ✅ After

```csharp
// Descriptive result types
public abstract record ValidationResult
{
    public sealed record Valid : ValidationResult;
    public sealed record Invalid(string Reason) : ValidationResult;
}

public abstract record LockStatus
{
    public sealed record Locked(DateTime Until, string Reason) : LockStatus;
    public sealed record Unlocked : LockStatus;
}

public class UserService
{
    public ValidationResult ValidateUser(string username, string password)
    {
        if (!_db.CheckCredentials(username, password))
            return new ValidationResult.Invalid("Incorrect username or password");
        
        return new ValidationResult.Valid();
    }

    public LockStatus GetLockStatus(int userId)
    {
        var lockInfo = _db.GetLockInfo(userId);
        if (lockInfo is null)
            return new LockStatus.Unlocked();
        
        return new LockStatus.Locked(lockInfo.Until, lockInfo.Reason);
    }
}

// Crystal clear at call site
var result = userService.ValidateUser(username, password);
switch (result)
{
    case ValidationResult.Valid:
        GrantAccess();
        break;
    case ValidationResult.Invalid invalid:
        ShowError(invalid.Reason);
        break;
}

// No ambiguity about lock status
var lockStatus = userService.GetLockStatus(userId);
if (lockStatus is LockStatus.Unlocked)
{
    AllowAccess();
}

// Replace boolean parameters with dedicated methods or types
// Option A: Separate methods for each case
public class OrderProcessor
{
    public void ProcessOrder(OrderId orderId, OrderOptions options)
    {
        // Single method with a typed options object
    }
    
    // Or dedicated methods that make intent clear
    public void ProcessStandardOrder(OrderId orderId) { /* ... */ }
    public void ProcessRushOrder(OrderId orderId) { /* ... */ }
}

// Option B: Configuration object
public sealed record OrderOptions
{
    public bool SendEmail { get; init; } = true;
    public bool ValidateStock { get; init; } = true;
    public bool Rush { get; init; } = false;
    
    public static OrderOptions Default => new();
    public static OrderOptions RushOrder => new() { Rush = true };
}

// Self-documenting call sites
processor.ProcessRushOrder(orderId);

processor.ProcessOrder(orderId, new OrderOptions 
{ 
    SendEmail = true, 
    Rush = true 
});

processor.ProcessOrder(orderId, OrderOptions.RushOrder);
```

## Why It's a Problem

1. **Invisible meaning**: `true` and `false` carry no semantic information at the call site.

2. **Easy to invert**: `if (!IsValid())` vs `if (IsValid())` — one character changes everything.

3. **No additional context**: A `bool` can't tell you *why* something is true or false.

4. **Boolean parameters**: `DoThing(true, false, true)` is incomprehensible without reading the signature.

## Symptoms

- Comments explaining what `true` and `false` mean
- Bugs caused by inverted boolean logic
- Method names like `IsNotEmpty()` to avoid double negatives
- Boolean parameters in method signatures
- `if (result == true)` instead of `if (result)` (developer unsure of meaning)
- Defensive naming like `successFlag` or `isValidResult`

## Alternatives

### Option 1: Dedicated Methods

Instead of boolean parameters, create separate methods with descriptive names:

```csharp
// Instead of
void SendNotification(UserId user, bool urgent);

// Create dedicated methods
void SendNotification(UserId user);
void SendUrgentNotification(UserId user);

// Or use a fluent builder
NotificationBuilder
    .To(user)
    .Urgent()
    .Send();
```

### Option 2: Configuration Objects

Group related options into a typed configuration:

```csharp
public sealed record ExportOptions
{
    public bool IncludeHeaders { get; init; } = true;
    public bool CompressOutput { get; init; } = false;
    public Encoding Encoding { get; init; } = Encoding.UTF8;
    
    public static ExportOptions Default => new();
    public static ExportOptions Compressed => new() { CompressOutput = true };
}

void Export(ReportId id, ExportOptions options);

// Usage
Export(reportId, ExportOptions.Default);
Export(reportId, ExportOptions.Compressed);
Export(reportId, new ExportOptions { IncludeHeaders = false });
```

### Option 3: Result Types

For operations that can fail, return a result type that carries context:

```csharp
public readonly record struct ParseResult<T>(bool Success, T? Value, string? Error)
{
    public static ParseResult<T> Ok(T value) => new(true, value, null);
    public static ParseResult<T> Fail(string error) => new(false, default, error);
}

public ParseResult<int> ParseAge(string input)
{
    if (int.TryParse(input, out var age) && age > 0 && age < 150)
        return ParseResult<int>.Ok(age);
    
    return ParseResult<int>.Fail($"'{input}' is not a valid age");
}
```

### Option 4: Named Booleans (Minimal Fix)

When refactoring isn't practical, at least name the boolean:

```csharp
// Instead of
if (Validate(order))

// Use a named variable
bool orderIsValid = Validate(order);
if (orderIsValid)

// Or use named arguments for boolean parameters (still a smell, but clearer)
ProcessOrder(orderId, sendEmail: true, priority: false, validateStock: true);
```

## The TryParse Pattern

.NET's `TryParse` pattern uses `bool` but mitigates blindness through convention:

```csharp
// The 'out' parameter makes success/failure clear
if (int.TryParse(input, out var value))
{
    // value is valid here
}
else
{
    // parsing failed
}
```

This works because:
- The pattern is universally understood
- The `out` parameter provides the value on success
- The method name starts with `Try`, signaling possible failure

## Benefits

- **Self-documenting**: Dedicated methods like `SendUrgentNotification()` are clearer than `Send(true)`
- **Additional context**: `Invalid("Password too short")` explains *why*
- **Impossible to invert**: Pattern matching forces handling both cases
- **IDE support**: Types and methods provide autocomplete and discoverability
- **Exhaustiveness checking**: Compiler warns if you miss a case

## See Also

- [Flag Arguments](./flag-arguments.md)
- [Honest Functions](./honest-functions.md)
- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md)
