# Nullable Reference Types

> Allowing `null` to flow through the codebase unchecked—enable nullable reference types and treat warnings as errors to catch null issues at compile time.

## Problem

Without nullable reference types enabled, C# treats all reference types as nullable by default. The compiler can't distinguish between "this might be null" and "this is never null," so null reference exceptions hide until runtime.

## Configuration

### Enable in Project File

```xml
<!-- In your .csproj -->
<PropertyGroup>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```

Or, if `TreatWarningsAsErrors` is too aggressive for your project, at minimum enable the null-related warnings:

```xml
<PropertyGroup>
    <Nullable>enable</Nullable>
    <WarningsAsErrors>CS8600;CS8601;CS8602;CS8603;CS8604;CS8625;CS8629</WarningsAsErrors>
</PropertyGroup>
```

### Warning Reference

| Code | Description |
|------|-------------|
| CS8600 | Converting null literal or possible null value to non-nullable type |
| CS8601 | Possible null reference assignment |
| CS8602 | Dereference of a possibly null reference |
| CS8603 | Possible null reference return |
| CS8604 | Possible null reference argument |
| CS8625 | Cannot convert null literal to non-nullable reference type |
| CS8629 | Nullable value type may be null |

## Example

### ❌ Before (Nullable Disabled)

```csharp
public class UserService
{
    public User GetUser(int id)
    {
        return _db.Users.FirstOrDefault(u => u.Id == id);
        // Returns null if not found, but signature says User
    }

    public string GetDisplayName(int id)
    {
        var user = GetUser(id);
        return user.DisplayName;  // NullReferenceException waiting to happen
    }
}
```

### ✅ After (Nullable Enabled)

```csharp
public class UserService
{
    public User? GetUser(int id)  // ? makes nullability explicit
    {
        return _db.Users.FirstOrDefault(u => u.Id == id);
    }

    public string GetDisplayName(int id)
    {
        var user = GetUser(id);
        return user.DisplayName;  // ❌ CS8602: Dereference of possibly null reference
    }

    // Fixed version
    public string GetDisplayNameSafe(int id)
    {
        var user = GetUser(id);
        if (user is null)
            return "Unknown";
        
        return user.DisplayName;  // ✅ Compiler knows user is not null here
    }
}
```

## Patterns Enabled by Nullable Reference Types

### Non-Nullable Parameters

```csharp
public void Process(User user)  // user cannot be null
{
    // No null check needed—compiler enforces at call site
    Console.WriteLine(user.Name);
}

Process(null);  // ❌ CS8625: Cannot convert null to non-nullable reference type
```

### Explicit Nullability

```csharp
public User? FindUser(int id);      // Might return null
public User GetUserOrThrow(int id); // Never returns null (throws if not found)
```

### Null-Forgiving Operator (Use Sparingly)

```csharp
// When you know better than the compiler
var user = GetUser(id)!;  // ! tells compiler "trust me, it's not null"

// Prefer this only at boundaries or after validation
```

### Required Members (C# 11+)

```csharp
public class User
{
    public required string Name { get; init; }  // Must be set, never null
    public string? Nickname { get; init; }      // Optional
}

var user = new User();           // ❌ CS9035: Required member 'Name' must be set
var user = new User { Name = "Alice" };  // ✅
```

## Migration Strategy

For existing projects, migrate incrementally:

### 1. Enable with Annotations Only

```xml
<Nullable>annotations</Nullable>
```

This adds the syntax without warnings—lets you annotate without breaking the build.

### 2. Enable Warnings

```xml
<Nullable>enable</Nullable>
```

Now warnings appear. Fix them gradually.

### 3. Treat as Errors

```xml
<Nullable>enable</Nullable>
<TreatWarningsAsErrors>true</TreatWarningsAsErrors>
```

Prevent regressions.

### Per-File Override

During migration, you can disable per-file:

```csharp
#nullable disable
// Legacy code here
#nullable restore
```

## Why This Matters

Many patterns in this catalog depend on nullable reference types:

- **[Capability Security](./capability-security.md)**: Prevents passing `null` instead of a valid capability token
- **[Strongly Typed IDs](./strongly-typed-ids.md)**: Ensures IDs can't be null unless explicitly marked
- **[Honest Functions](./honest-functions.md)**: `User?` vs `User` tells the truth about return values
- **[Nullability vs. Optionality](./nullability-optionality.md)**: Foundation for distinguishing null from Option

Without this enabled, these patterns can be circumvented by passing `null`.

## Benefits

- **Compile-time null safety**: Catch null issues before runtime
- **Self-documenting APIs**: `User?` vs `User` is part of the signature
- **Reduced defensive coding**: No need for null checks on non-nullable parameters
- **IDE support**: Warnings appear as you type, not in production logs
- **Foundation for type-safe patterns**: Enables other patterns to rely on non-null guarantees

## See Also

- [Nullability vs. Optionality](./nullability-optionality.md) — when to use `Option<T>` vs `T?`
- [Honest Functions](./honest-functions.md) — making signatures truthful
- [Capability Security](./capability-security.md) — depends on non-nullable enforcement
