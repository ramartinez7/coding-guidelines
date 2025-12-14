# Compile-Time Validation (Catching Errors Before Runtime)

> Runtime validation catches errors in production—move validation to compile time using types to catch bugs during development.

## Problem

Validation that happens at runtime allows invalid code to compile. Bugs slip through to production because the compiler can't verify business rules encoded as runtime checks.

## Example

### ❌ Before

```csharp
// Runtime validation
public class UserService
{
    public Result<User, string> CreateUser(
        string email,
        string password,
        int age)
    {
        // All validation happens at runtime
        if (string.IsNullOrWhiteSpace(email))
            return Result<User, string>.Failure("Email required");
        
        if (!email.Contains("@"))
            return Result<User, string>.Failure("Invalid email");
        
        if (password.Length < 8)
            return Result<User, string>.Failure("Password too short");
        
        if (age < 18)
            return Result<User, string>.Failure("Must be 18+");
        
        if (age > 150)
            return Result<User, string>.Failure("Invalid age");
        
        var user = new User
        {
            Email = email,
            Password = password,  // Stored in plain text!
            Age = age
        };
        
        return Result<User, string>.Success(user);
    }
}

// All of these compile but are invalid
var service = new UserService();
service.CreateUser("", "", -5);  // Compiles!
service.CreateUser("notanemail", "123", 200);  // Compiles!

// Forgot to validate before using
var user = new User
{
    Email = "invalid",  // Compiles!
    Password = "weak",  // Compiles!
    Age = -10  // Compiles!
};
```

**Problems:**
- Invalid values compile successfully
- Must remember to validate everywhere
- Easy to forget validation before using data
- Validation logic duplicated across codebase
- Compiler provides no help

### ✅ After

```csharp
// Validated types enforce rules at construction
public sealed record EmailAddress
{
    public string Value { get; }
    
    private EmailAddress(string value) => Value = value;
    
    public static Result<EmailAddress, string> Create(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            return Result<EmailAddress, string>.Failure("Email required");
        
        if (!email.Contains("@"))
            return Result<EmailAddress, string>.Failure("Invalid email format");
        
        if (email.Length > 255)
            return Result<EmailAddress, string>.Failure("Email too long");
        
        var normalized = email.Trim().ToLowerInvariant();
        
        return Result<EmailAddress, string>.Success(new EmailAddress(normalized));
    }
}

public sealed record HashedPassword
{
    public string Hash { get; }
    
    private HashedPassword(string hash) => Hash = hash;
    
    public static Result<HashedPassword, string> FromPlaintext(string plaintext)
    {
        if (string.IsNullOrWhiteSpace(plaintext))
            return Result<HashedPassword, string>.Failure("Password required");
        
        if (plaintext.Length < 8)
            return Result<HashedPassword, string>.Failure("Password must be at least 8 characters");
        
        if (plaintext.Length > 128)
            return Result<HashedPassword, string>.Failure("Password too long");
        
        if (!ContainsUppercase(plaintext))
            return Result<HashedPassword, string>.Failure("Password must contain uppercase letter");
        
        if (!ContainsLowercase(plaintext))
            return Result<HashedPassword, string>.Failure("Password must contain lowercase letter");
        
        if (!ContainsDigit(plaintext))
            return Result<HashedPassword, string>.Failure("Password must contain digit");
        
        var hash = BCrypt.Net.BCrypt.HashPassword(plaintext);
        return Result<HashedPassword, string>.Success(new HashedPassword(hash));
    }
    
    public bool Verify(string plaintext) =>
        BCrypt.Net.BCrypt.Verify(plaintext, Hash);
    
    private static bool ContainsUppercase(string s) => s.Any(char.IsUpper);
    private static bool ContainsLowercase(string s) => s.Any(char.IsLower);
    private static bool ContainsDigit(string s) => s.Any(char.IsDigit);
}

public sealed record Age
{
    public int Value { get; }
    
    private Age(int value) => Value = value;
    
    public static Result<Age, string> Create(int age)
    {
        if (age < 0)
            return Result<Age, string>.Failure("Age cannot be negative");
        
        if (age > 150)
            return Result<Age, string>.Failure("Age unrealistic");
        
        return Result<Age, string>.Success(new Age(age));
    }
    
    public static Result<Age, string> CreateAdult(int age)
    {
        var result = Create(age);
        
        if (!result.IsSuccess)
            return result;
        
        if (age < 18)
            return Result<Age, string>.Failure("Must be 18 or older");
        
        return result;
    }
}

// User entity accepts only validated types
public sealed class User
{
    public UserId Id { get; }
    public EmailAddress Email { get; }
    public HashedPassword Password { get; }
    public Age Age { get; }
    
    private User(UserId id, EmailAddress email, HashedPassword password, Age age)
    {
        Id = id;
        Email = email;
        Password = password;
        Age = age;
    }
    
    public static User Create(UserId id, EmailAddress email, HashedPassword password, Age age) =>
        new User(id, email, password, age);
}

// Service accepts only validated types
public sealed class UserService
{
    public async Task<User> CreateUser(
        EmailAddress email,  // Compiler enforces validation!
        HashedPassword password,  // Can't pass plain text!
        Age age)  // Must be valid age!
    {
        // No validation needed—types guarantee validity
        var user = User.Create(UserId.New(), email, password, age);
        
        await _userRepo.SaveAsync(user);
        
        return user;
    }
}

// Usage: Validation required to get validated types
var emailResult = EmailAddress.Create("john@example.com");
var passwordResult = HashedPassword.FromPlaintext("SecureP@ss123");
var ageResult = Age.CreateAdult(25);

// ✗ These don't compile:
// var user = new User(id, "invalid", "weak", -10);  // Type mismatch!
// service.CreateUser("email", "password", 25);  // Type mismatch!

// ✓ This compiles and is guaranteed valid:
if (emailResult.IsSuccess && passwordResult.IsSuccess && ageResult.IsSuccess)
{
    var user = await service.CreateUser(
        emailResult.Value,
        passwordResult.Value,
        ageResult.Value);
}
```

## Compile-Time Constraint Examples

### Example 1: Non-Empty Collections

```csharp
// Runtime: Can be empty
public class ShoppingCart
{
    public List<CartItem> Items { get; set; }
    
    public Result<Order, string> Checkout()
    {
        if (Items.Count == 0)  // Runtime check
            return Result<Order, string>.Failure("Cart is empty");
        
        // ...
    }
}

// ✅ Compile-time: Cannot be empty
public sealed record NonEmptyList<T>
{
    private readonly List<T> _items;
    public IReadOnlyList<T> Items => _items.AsReadOnly();
    public T First => _items[0];  // Safe—always has items
    public int Count => _items.Count;
    
    private NonEmptyList(T first, List<T> rest)
    {
        _items = new List<T> { first };
        _items.AddRange(rest);
    }
    
    public static NonEmptyList<T> Create(T first, params T[] rest) =>
        new NonEmptyList<T>(first, rest.ToList());
    
    public static Result<NonEmptyList<T>, string> FromList(List<T> items)
    {
        if (items.Count == 0)
            return Result<NonEmptyList<T>, string>.Failure("List cannot be empty");
        
        return Result<NonEmptyList<T>, string>.Success(
            new NonEmptyList<T>(items[0], items.Skip(1).ToList()));
    }
    
    public NonEmptyList<T> Add(T item)
    {
        var newList = _items.ToList();
        newList.Add(item);
        return new NonEmptyList<T>(newList[0], newList.Skip(1).ToList());
    }
}

public sealed class ShoppingCart
{
    public Option<NonEmptyList<CartItem>> Items { get; private set; }
    
    public Result<Order, string> Checkout()
    {
        return Items.Match(
            onSome: items => CreateOrder(items),  // Guaranteed non-empty!
            onNone: () => Result<Order, string>.Failure("Cart is empty"));
    }
    
    private Result<Order, string> CreateOrder(NonEmptyList<CartItem> items)
    {
        // No need to check if empty—type guarantees it
        var total = CalculateTotal(items);
        return Result<Order, string>.Success(Order.Create(items, total));
    }
}
```

### Example 2: Positive Numbers

```csharp
// Runtime: Can be negative
public class Product
{
    public decimal Price { get; set; }
    
    public Result<Unit, string> UpdatePrice(decimal newPrice)
    {
        if (newPrice < 0)  // Runtime check
            return Result<Unit, string>.Failure("Price cannot be negative");
        
        Price = newPrice;
        return Result<Unit, string>.Success(Unit.Value);
    }
}

// ✅ Compile-time: Cannot be negative
public readonly record struct PositiveDecimal
{
    public decimal Value { get; }
    
    private PositiveDecimal(decimal value) => Value = value;
    
    public static Result<PositiveDecimal, string> Create(decimal value)
    {
        if (value <= 0)
            return Result<PositiveDecimal, string>.Failure("Value must be positive");
        
        return Result<PositiveDecimal, string>.Success(new PositiveDecimal(value));
    }
    
    public static PositiveDecimal operator +(PositiveDecimal a, PositiveDecimal b) =>
        new PositiveDecimal(a.Value + b.Value);  // Result always positive
    
    public static PositiveDecimal operator *(PositiveDecimal a, PositiveDecimal b) =>
        new PositiveDecimal(a.Value * b.Value);  // Result always positive
}

public sealed class Product
{
    public ProductId Id { get; }
    public ProductName Name { get; }
    public Money Price { get; private set; }  // Money contains PositiveDecimal
    
    public void UpdatePrice(Money newPrice)
    {
        // No validation needed—Money type guarantees positive amount
        Price = newPrice;
    }
}
```

### Example 3: Valid Date Ranges

```csharp
// Runtime: End can be before start
public class Event
{
    public DateTime Start { get; set; }
    public DateTime End { get; set; }
    
    public TimeSpan Duration
    {
        get
        {
            if (End < Start)  // Runtime check
                throw new InvalidOperationException("End before start");
            
            return End - Start;
        }
    }
}

// ✅ Compile-time: End always after start
public sealed record DateRange
{
    public DateTimeOffset Start { get; }
    public DateTimeOffset End { get; }
    public TimeSpan Duration => End - Start;  // Always positive!
    
    private DateRange(DateTimeOffset start, DateTimeOffset end)
    {
        Start = start;
        End = end;
    }
    
    public static Result<DateRange, string> Create(DateTimeOffset start, DateTimeOffset end)
    {
        if (end <= start)
            return Result<DateRange, string>.Failure("End must be after start");
        
        return Result<DateRange, string>.Success(new DateRange(start, end));
    }
    
    public bool Contains(DateTimeOffset point) =>
        point >= Start && point <= End;
    
    public bool OverlapsWith(DateRange other) =>
        Start < other.End && other.Start < End;
}

public sealed class Event
{
    public EventId Id { get; }
    public EventName Name { get; }
    public DateRange Schedule { get; }
    
    private Event(EventId id, EventName name, DateRange schedule)
    {
        Id = id;
        Name = name;
        Schedule = schedule;
    }
    
    public static Event Create(EventId id, EventName name, DateRange schedule) =>
        new Event(id, name, schedule);
    
    public bool IsActive(DateTimeOffset now) =>
        Schedule.Contains(now);
}
```

### Example 4: Constrained String Lengths

```csharp
// ✅ Compile-time: String within length bounds
public readonly record struct BoundedString
{
    public string Value { get; }
    public int MinLength { get; }
    public int MaxLength { get; }
    
    private BoundedString(string value, int minLength, int maxLength)
    {
        Value = value;
        MinLength = minLength;
        MaxLength = maxLength;
    }
    
    public static Result<BoundedString, string> Create(
        string value,
        int minLength,
        int maxLength)
    {
        if (string.IsNullOrEmpty(value))
            return Result<BoundedString, string>.Failure("Value required");
        
        if (value.Length < minLength)
            return Result<BoundedString, string>.Failure(
                $"Value too short (min {minLength})");
        
        if (value.Length > maxLength)
            return Result<BoundedString, string>.Failure(
                $"Value too long (max {maxLength})");
        
        return Result<BoundedString, string>.Success(
            new BoundedString(value, minLength, maxLength));
    }
}

// Domain-specific bounded strings
public sealed record Username
{
    public BoundedString Value { get; }
    
    private Username(BoundedString value) => Value = value;
    
    public static Result<Username, string> Create(string username)
    {
        var result = BoundedString.Create(username, 3, 20);
        
        if (!result.IsSuccess)
            return Result<Username, string>.Failure(result.Error);
        
        if (!IsValidFormat(username))
            return Result<Username, string>.Failure(
                "Username must contain only letters, numbers, and underscores");
        
        return Result<Username, string>.Success(new Username(result.Value));
    }
    
    private static bool IsValidFormat(string s) =>
        s.All(c => char.IsLetterOrDigit(c) || c == '_');
}

public sealed record Bio
{
    public BoundedString Value { get; }
    
    private Bio(BoundedString value) => Value = value;
    
    public static Result<Bio, string> Create(string bio)
    {
        var result = BoundedString.Create(bio, 10, 500);
        
        return result.IsSuccess
            ? Result<Bio, string>.Success(new Bio(result.Value))
            : Result<Bio, string>.Failure(result.Error);
    }
}
```

## Why It's a Problem

1. **Runtime errors**: Invalid data compiles but fails in production
2. **Forgotten validation**: Easy to forget checks before using data
3. **Duplicated logic**: Same validation repeated everywhere
4. **No compiler help**: Type system can't catch validation errors
5. **Testing burden**: Must test validation at every usage point

## Symptoms

- Validation logic scattered throughout codebase
- Same validation checks repeated in multiple places
- Runtime exceptions from invalid data
- Primitive types used for domain concepts
- Method parameters validated at start of every method

## Benefits

- **Early error detection**: Catch bugs at compile time
- **Single validation point**: Validate once at construction
- **Compiler assistance**: Type system prevents invalid code
- **Self-documenting**: Types reveal constraints
- **Reduced testing**: Don't test validation everywhere

## See Also

- [Smart Constructors](./smart-constructors.md) — parse, don't validate
- [Domain Invariants](./domain-invariants.md) — enforcing business rules
- [Refinement Types](./refinement-types.md) — types with constraints
- [Branded Primitives](./branded-primitives.md) — nominal typing
- [Making Invalid States Unrepresentable](./making-invalid-states-unrepresentable.md) — core principle
