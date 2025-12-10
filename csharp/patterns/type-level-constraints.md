# Type-Level Constraints (Encoding Constraints in the Type System)

> Runtime validation catches errors late—encode constraints in types to catch violations at compile time.

## Problem

Constraints validated at runtime can be violated, and the compiler can't help prevent mistakes.

## Example

### ❌ Runtime Constraints

```csharp
public class Email
{
    public string Value { get; set; }
    
    public bool IsValid() => Value?.Contains("@") == true;
}

var email = new Email { Value = "invalid" };  // Compiles!
if (!email.IsValid()) { }  // Must remember to check
```

### ✅ Type-Level Constraints

```csharp
// Constraint: Email must contain @ symbol
public sealed record EmailAddress
{
    public string Value { get; }
    
    private EmailAddress(string value) => Value = value;
    
    public static Result<EmailAddress, string> Create(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            return Result<EmailAddress, string>.Failure("Email required");
        
        if (!email.Contains("@"))
            return Result<EmailAddress, string>.Failure("Invalid email");
        
        return Result<EmailAddress, string>.Success(new EmailAddress(email));
    }
}

// EmailAddress type guarantees constraint
public void SendEmail(EmailAddress to)
{
    // No validation needed—type guarantees it's valid
}
```

## More Examples

```csharp
// Constraint: NonZero integer
public readonly record struct NonZeroInt
{
    public int Value { get; }
    
    private NonZeroInt(int value) => Value = value;
    
    public static Result<NonZeroInt, string> Create(int value)
    {
        if (value == 0)
            return Result<NonZeroInt, string>.Failure("Value cannot be zero");
        
        return Result<NonZeroInt, string>.Success(new NonZeroInt(value));
    }
}

// Constraint: List with at least one element
public sealed record NonEmptyList<T>
{
    private readonly List<T> _items;
    public T Head => _items[0];  // Safe—always has elements
    public IReadOnlyList<T> Tail => _items.Skip(1).ToList();
    
    private NonEmptyList(List<T> items) => _items = items;
    
    public static NonEmptyList<T> Create(T first, params T[] rest)
    {
        var items = new List<T> { first };
        items.AddRange(rest);
        return new NonEmptyList<T>(items);
    }
}
```

## See Also

- [Refinement Types](./refinement-types.md)
- [Smart Constructors](./smart-constructors.md)
- [Compile-Time Validation](./compile-time-validation.md)
