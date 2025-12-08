# When to Create Domain Types (Domain Primitive Decision Guide)

> Not every string, int, or decimal needs its own type—use this guide to decide when to create domain-specific types versus using primitives.

## Problem

Creating too many types adds ceremony without value. Using too few types loses type safety and domain clarity. This guide helps find the right balance based on business value, invariants, and operations.

## Decision Framework

### ✅ Create a Domain Type When:

1. **Has Business Meaning** — The value represents a domain concept
2. **Has Invariants** — Rules constrain valid values
3. **Has Operations** — Domain-specific behavior beyond storage
4. **Prevents Errors** — Type system catches common mistakes
5. **Used Across Boundaries** — Passed between layers or services
6. **Has Multiple Representations** — Formatting, serialization varies by context

### ❌ Use Primitive When:

1. **Pure Data** — No behavior, just storage
2. **Single Use** — Only in one place, not reused
3. **No Invariants** — Any value is valid
4. **Simple Display** — No special formatting
5. **Internal Only** — Never crosses boundaries

## Examples by Category

### Category 1: Identifiers

| Concept | Create Type? | Reason |
|---------|--------------|--------|
| `CustomerId` | ✅ Yes | Prevents ID swapping, domain concept |
| `OrderId` | ✅ Yes | Prevents ID swapping, domain concept |
| `Internal array index` | ❌ No | Implementation detail, local to method |
| `Loop counter` | ❌ No | Pure mechanics, no domain meaning |

```csharp
// ✅ Create type: Domain identifier
public readonly record struct CustomerId(Guid Value);
public readonly record struct OrderId(Guid Value);

// ❌ Don't create type: Implementation detail
public void ProcessBatch(Order[] orders)
{
    for (int i = 0; i < orders.Length; i++)  // Just use int
    {
        ProcessOrder(orders[i]);
    }
}
```

### Category 2: Constrained Values

| Concept | Create Type? | Reason |
|---------|--------------|--------|
| `Email` | ✅ Yes | Has format rules, validation needed |
| `PhoneNumber` | ✅ Yes | Has format rules, country variations |
| `Age` | ✅ Yes | Has range constraints (0-150) |
| `Percentage` | ✅ Yes | Has range (0-100), decimal vs integer confusion |
| `Configuration value` | ❌ No | Already validated at startup, internal only |

```csharp
// ✅ Create type: Has validation rules
public readonly record struct Age
{
    public int Value { get; }
    
    private Age(int value) => Value = value;
    
    public static Result<Age, string> Create(int value)
    {
        if (value < 0 || value > 150)
            return Result<Age, string>.Failure("Age must be 0-150");
        
        return Result<Age, string>.Success(new Age(value));
    }
}

// ❌ Don't create type: Already validated configuration
public class AppSettings
{
    public int MaxRetries { get; init; }  // Validated at startup
    public int TimeoutSeconds { get; init; }
}
```

### Category 3: Money and Measurements

| Concept | Create Type? | Reason |
|---------|--------------|--------|
| `Money` | ✅ Yes | Currency + amount, prevents mixing currencies |
| `Distance` | ✅ Yes | Unit conversion (meters/feet), prevents errors |
| `Temperature` | ✅ Yes | Unit conversion (C/F), domain operations |
| `Weight` | ✅ Yes | Unit conversion, prevents mixing units |
| `Database byte size` | ❌ Maybe | If just displaying, use long; if converting units, create type |

```csharp
// ✅ Create type: Prevents mixing currencies
public readonly record struct Money(decimal Amount, Currency Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return new Money(Amount + other.Amount, Currency);
    }
}

// ✅ Create type: Unit conversion and operations
public readonly record struct Distance
{
    private readonly decimal _meters;
    
    private Distance(decimal meters) => _meters = meters;
    
    public static Distance FromMeters(decimal meters) => new(meters);
    public static Distance FromKilometers(decimal km) => new(km * 1000);
    public static Distance FromMiles(decimal miles) => new(miles * 1609.344m);
    
    public decimal ToMeters() => _meters;
    public decimal ToKilometers() => _meters / 1000;
    public decimal ToMiles() => _meters / 1609.344m;
    
    public static Distance operator +(Distance a, Distance b) =>
        new(a._meters + b._meters);
}

// ❌ Don't create type: Simple storage, no operations
public sealed record CacheEntry
{
    public string Key { get; init; }
    public long SizeBytes { get; init; }  // Just a number, no operations
    public DateTimeOffset CreatedAt { get; init; }
}
```

### Category 4: Formatted Strings

| Concept | Create Type? | Reason |
|---------|--------------|--------|
| `EmailAddress` | ✅ Yes | Validation, prevents typos in methods |
| `PhoneNumber` | ✅ Yes | Formatting, international variations |
| `PostalCode` | ✅ Yes | Format rules, validation |
| `Url` | ✅ Yes | Validation, serialization |
| `Log message` | ❌ No | Free-form text, no invariants |
| `User comment` | ❌ No | Free-form text, no validation |

```csharp
// ✅ Create type: Has format and validation
public sealed record EmailAddress
{
    private const string Pattern = @"^[^@\s]+@[^@\s]+\.[^@\s]+$";
    
    public string Value { get; }
    
    private EmailAddress(string value) => Value = value;
    
    public static Result<EmailAddress, string> Parse(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return Result<EmailAddress, string>.Failure("Email cannot be empty");
        
        if (!Regex.IsMatch(input, Pattern))
            return Result<EmailAddress, string>.Failure("Invalid email format");
        
        return Result<EmailAddress, string>.Success(new EmailAddress(input));
    }
}

// ❌ Don't create type: Free-form text
public sealed record Comment
{
    public string Text { get; init; }  // Just use string
    public UserId AuthorId { get; init; }
    public DateTimeOffset CreatedAt { get; init; }
}
```

### Category 5: Dates and Times

| Concept | Create Type? | Reason |
|---------|--------------|--------|
| `DateRange` | ✅ Yes | Enforces end >= start, has operations (overlap, contains) |
| `BusinessHours` | ✅ Yes | Complex rules, time zone handling |
| `Event timestamp` | ❌ No | Use `DateTimeOffset`, no special rules |
| `Log timestamp` | ❌ No | Use `DateTimeOffset`, pure data |

```csharp
// ✅ Create type: Has invariants and operations
public sealed record DateRange
{
    public DateOnly Start { get; }
    public DateOnly End { get; }
    
    private DateRange(DateOnly start, DateOnly end)
    {
        Start = start;
        End = end;
    }
    
    public static Result<DateRange, string> Create(DateOnly start, DateOnly end)
    {
        if (end < start)
            return Result<DateRange, string>.Failure("End must be >= start");
        
        return Result<DateRange, string>.Success(new DateRange(start, end));
    }
    
    public int Days => End.DayNumber - Start.DayNumber + 1;
    public bool Contains(DateOnly date) => date >= Start && date <= End;
    public bool Overlaps(DateRange other) => Start <= other.End && End >= other.Start;
}

// ❌ Don't create type: Pure timestamp
public sealed record AuditLogEntry
{
    public string Action { get; init; }
    public UserId UserId { get; init; }
    public DateTimeOffset Timestamp { get; init; }  // Just use DateTimeOffset
}
```

### Category 6: Collections

| Concept | Create Type? | Reason |
|---------|--------------|--------|
| `NonEmptyList<T>` | ✅ Yes | Enforces invariant, prevents errors |
| `UniqueList<T>` | ✅ Yes | Enforces no duplicates |
| `OrderLines` (aggregate) | ✅ Yes | Domain operations (add, remove, total) |
| `Query results` | ❌ No | Use `IReadOnlyList<T>`, pure data |

```csharp
// ✅ Create type: Enforces invariant
public sealed record NonEmptyList<T>
{
    private readonly List<T> _items;
    
    public T First { get; }
    public IReadOnlyList<T> All => _items.AsReadOnly();
    
    private NonEmptyList(T first, IEnumerable<T> rest)
    {
        First = first;
        _items = new List<T> { first };
        _items.AddRange(rest);
    }
    
    public static Result<NonEmptyList<T>, string> FromList(IEnumerable<T> items)
    {
        var list = items.ToList();
        if (list.Count == 0)
            return Result<NonEmptyList<T>, string>.Failure("List cannot be empty");
        
        return Result<NonEmptyList<T>, string>.Success(
            new NonEmptyList<T>(list[0], list.Skip(1)));
    }
}

// ❌ Don't create type: Pure data transfer
public sealed record SearchResults
{
    public IReadOnlyList<Product> Products { get; init; }  // Just use interface
    public int TotalCount { get; init; }
    public int Page { get; init; }
}
```

## Decision Tree

```
Does the value represent a domain concept?
├─ No → Use primitive
└─ Yes → Does it have invariants/rules?
    ├─ No → Does it have domain operations?
    │   ├─ No → Use primitive
    │   └─ Yes → CREATE TYPE
    └─ Yes → Does it cross layer boundaries?
        ├─ No → Maybe use primitive (if local scope)
        └─ Yes → CREATE TYPE
```

## Real-World Examples

### Example 1: E-Commerce Order

```csharp
public sealed class Order
{
    // ✅ Create types: Domain identifiers
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    
    // ✅ Create type: Has operations and invariants
    public Money Total { get; private set; }
    
    // ✅ Create type: Has format and validation
    public EmailAddress CustomerEmail { get; }
    
    // ❌ Use primitive: Pure data, no invariants
    public string InternalNotes { get; private set; }  // Free-form text
    
    // ❌ Use primitive: Already validated by type system
    public DateTimeOffset CreatedAt { get; }
    
    // ✅ Create type: Collection with invariants
    private readonly NonEmptyList<OrderLine> _items;
}
```

### Example 2: User Profile

```csharp
public sealed class UserProfile
{
    // ✅ Create type: Domain identifier
    public UserId Id { get; }
    
    // ✅ Create type: Validation rules
    public EmailAddress Email { get; private set; }
    
    // ✅ Create type: Validation rules
    public PhoneNumber PhoneNumber { get; private set; }
    
    // ❌ Use primitive: No special rules
    public string DisplayName { get; private set; }  // Any string valid
    
    // ❌ Use primitive: Free-form text
    public string Bio { get; private set; }
    
    // ✅ Create type: Has validation
    public Age Age { get; private set; }
    
    // ❌ Use primitive: Pure timestamp
    public DateTimeOffset LastLoginAt { get; private set; }
    
    // ✅ Create type: Constrained value
    public ProfileVisibility Visibility { get; private set; }  // Enum or class
}
```

### Example 3: Financial Transaction

```csharp
public sealed class Transaction
{
    // ✅ Create type: Domain identifier
    public TransactionId Id { get; }
    
    // ✅ Create type: Prevents currency mixing
    public Money Amount { get; }
    
    // ✅ Create type: Domain identifiers
    public AccountId FromAccount { get; }
    public AccountId ToAccount { get; }
    
    // ❌ Use primitive: Free-form text
    public string Description { get; }
    
    // ❌ Use primitive: Pure timestamp
    public DateTimeOffset ProcessedAt { get; }
    
    // ✅ Create type: Has format and validation
    public TransactionReference ReferenceNumber { get; }
    
    // ✅ Create type: Workflow states
    public TransactionStatus Status { get; private set; }  // Type-safe state machine
}
```

## Cost-Benefit Analysis

### High Value Domain Types

**Create when:**
- Prevents parameter swapping errors
- Enforces business invariants
- Has domain-specific operations
- Used in multiple places
- Crosses layer boundaries

**Examples:**
- `CustomerId`, `OrderId` — prevents ID mixing
- `Money` — prevents currency errors
- `EmailAddress` — validation at boundary
- `DateRange` — enforces end >= start

### Low Value Domain Types

**Avoid when:**
- Pure data with no behavior
- Single use, local scope
- No invariants to enforce
- Simple display/storage
- Would add ceremony without benefit

**Examples:**
- Loop counters
- Temporary calculations
- Free-form text fields
- Already-validated config values
- Pure timestamps

## Guidelines Summary

1. **Identity** → Always create type (prevents swapping)
2. **Constrained value** → Create type (enforce invariants)
3. **Money/measurements** → Create type (prevent unit mixing)
4. **Formatted string** → Create type if validated
5. **Free-form text** → Use primitive
6. **Timestamp** → Use primitive (unless special rules)
7. **Configuration** → Use primitive (validate at startup)
8. **Collection** → Create type if has invariants

## Benefits of Right-Sized Types

- **Clarity**: Domain concepts explicit in code
- **Safety**: Type system prevents errors
- **Maintainability**: Changes localized to type
- **Not Over-Engineered**: Ceremony only where it adds value

## See Also

- [Primitive Obsession](./primitive-obsession.md) — when primitives are overused
- [Strongly Typed IDs](./strongly-typed-ids.md) — identity types
- [Value Semantics](./value-semantics.md) — value object implementation
- [Domain Invariants](./domain-invariants.md) — enforcing business rules
- [Smart Constructors](./smart-constructors.md) — validation patterns
