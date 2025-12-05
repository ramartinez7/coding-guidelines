# Primitive Obsession

> Over-use of "primitive" built-in types instead of domain-specific types.

## Problem

Using primitives like `string` or `int` for domain concepts loses meaning and safety. Validation logic gets scattered and duplicated, call sites become ambiguous, and rule changes require codebase-wide edits.

## Example

### ❌ Before

```csharp
public sealed class UserService
{
    public void RegisterUser(string email, string displayName)
    {
        if (string.IsNullOrWhiteSpace(email) || !email.Contains("@"))
            throw new ArgumentException("Invalid email", nameof(email));

        Console.WriteLine($"Welcome {displayName}! We'll email you at {email.Trim().ToLowerInvariant()}.");
    }
}
```

### ✅ After

```csharp
public readonly record struct EmailAddress
{
    public string Value { get; }

    private EmailAddress(string value)
    {
        Value = value;
    }

    public static EmailAddress FromString(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains("@"))
            throw new ArgumentException("Invalid email", nameof(value));

        return new EmailAddress(value.Trim().ToLowerInvariant());
    }

    public static bool TryFromString(string value, out EmailAddress result)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains("@"))
        {
            result = default;
            return false;
        }
        result = new EmailAddress(value.Trim().ToLowerInvariant());
        return true;
    }

    public static Result<EmailAddress> Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains("@"))
            return Result.Failure<EmailAddress>("Invalid email format");

        return Result.Success(new EmailAddress(value.Trim().ToLowerInvariant()));
    }
}

public sealed class UserService
{
    public void RegisterUser(EmailAddress email, string displayName)
    {
        Console.WriteLine($"Welcome {displayName}! We'll email you at {email.Value}.");
    }
}

// Usage:
var email = EmailAddress.FromString("user@example.com");  // Throws if invalid

if (EmailAddress.TryFromString(userInput, out var validEmail))  // TryParse pattern
{
    service.RegisterUser(validEmail, "Alice");
}

var result = EmailAddress.Create(userInput);  // Result pattern
if (result.IsSuccess)
{
    service.RegisterUser(result.Value, "Bob");
}
else
{
    logger.LogWarning("Registration failed: {Error}", result.Error);
}
```

## Why It's a Problem

1. **Lost domain meaning**: Any string looks valid to the compiler; runtime accepts garbage that passes naive checks.
2. **Scattered invariants**: Validation/normalization repeated inconsistently across modules.
3. **Ambiguity**: Call sites don't communicate intent—is this a username, ID, or email?
4. **Maintenance cost**: Rule changes require codebase-wide edits (easy to miss one).

## Symptoms

- Built-in types (`string`, `int`, etc.) in method signatures where domain types would be more appropriate
- Copy/pasted value validation across multiple locations
- Value validation outside of a constructor
- Asking yourself: Do I really need a string 2 billion codepoints long? Negative numbers? 10 decimal places?

## Benefits

- **Meaningful types** communicate intent clearly
- **Single source of truth** for validation and formatting
- **Safer APIs** prevent misuse at compile time

## See Also

- [Data Clumps](./data-clump.md)
- [Static Factory Methods](./static-factory-methods.md)
