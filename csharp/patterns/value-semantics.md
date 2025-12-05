# Value Semantics

> Using reference types with mutable state for concepts that should be values—use records or structs instead.

## Problem

When domain concepts represent values (like money, dates, coordinates), using mutable reference types leads to aliasing bugs, broken equality, and defensive copying. Values should be compared by their content, not their memory address.

## Example

### ❌ Before

```csharp
public class Money
{
    public decimal Amount { get; set; }
    public string Currency { get; set; }
}

// Aliasing bug
var price = new Money { Amount = 100, Currency = "USD" };
var discount = price;  // Both point to same object!
discount.Amount = 80;
Console.WriteLine(price.Amount);  // 80 — price was mutated!

// Broken equality
var a = new Money { Amount = 100, Currency = "USD" };
var b = new Money { Amount = 100, Currency = "USD" };
Console.WriteLine(a == b);  // False — different references
Console.WriteLine(a.Equals(b));  // False — default reference equality

// Dictionary key disaster
var prices = new Dictionary<Money, string>();
var key = new Money { Amount = 100, Currency = "USD" };
prices[key] = "Product A";
key.Amount = 200;  // Mutating the key!
Console.WriteLine(prices.ContainsKey(key));  // False — hash code changed
```

### ✅ After

```csharp
public readonly record struct Money(decimal Amount, Currency Currency)
{
    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return new Money(a.Amount + b.Amount, a.Currency);
    }

    public Money WithAmount(decimal newAmount) => this with { Amount = newAmount };
}

public readonly record struct Currency(string Code)
{
    public static Currency USD => new("USD");
    public static Currency EUR => new("EUR");
}

// No aliasing — value is copied
var price = new Money(100, Currency.USD);
var discount = price with { Amount = 80 };  // New instance
Console.WriteLine(price.Amount);  // 100 — original unchanged

// Structural equality works
var a = new Money(100, Currency.USD);
var b = new Money(100, Currency.USD);
Console.WriteLine(a == b);  // True — same values

// Safe as dictionary keys (immutable)
var prices = new Dictionary<Money, string>
{
    [new Money(100, Currency.USD)] = "Product A"
};
Console.WriteLine(prices.ContainsKey(new Money(100, Currency.USD)));  // True
```

## Why It's a Problem

1. **Aliasing bugs**: Multiple variables pointing to the same object cause unexpected mutations.

2. **Broken equality**: Reference equality means two identical values are considered "different".

3. **Unsafe dictionary keys**: Mutable objects as keys break hash-based collections when mutated.

4. **Defensive copying**: Callers must manually copy objects to prevent mutation—error-prone and verbose.

## Symptoms

- Cloning or copying objects defensively before passing them
- Overriding `Equals` and `GetHashCode` on every class
- Bugs where "changing one thing changed something else"
- Comments warning "don't mutate this" or "this is a copy"
- Domain concepts with setters that shouldn't change after creation

## When to Use Value Semantics

| Concept | Value Type? | Example |
|---------|-------------|---------|
| Money | ✅ Yes | `$100 USD` is a value |
| Coordinates | ✅ Yes | `(lat: 40.7, lon: -74.0)` |
| Date ranges | ✅ Yes | `Jan 1 - Jan 31` |
| Email address | ✅ Yes | `user@example.com` |
| User entity | ❌ No | Has identity, lifecycle |
| Order entity | ❌ No | Tracks state changes |
| Database connection | ❌ No | Resource with identity |

## Choosing the Right Type

| Type | Stack/Heap | Equality | Best For |
|------|------------|----------|----------|
| `readonly record struct` | Stack | Structural | Small values (≤16 bytes), high frequency |
| `record` (class) | Heap | Structural | Larger values, inheritance needed |
| `struct` | Stack | Manual | Interop, performance-critical |
| `class` | Heap | Reference | Entities with identity |

## Benefits

- **No aliasing**: Values are copied, not shared
- **Built-in equality**: Records provide structural equality automatically
- **Safe dictionary keys**: Immutable values have stable hash codes
- **Thread-safe by default**: Immutable values can't have race conditions
- **Predictable behavior**: What you see is what you get

## Variation: Immutable Class (When Struct Won't Work)

```csharp
public sealed record DateRange(DateOnly Start, DateOnly End)
{
    public DateRange(DateOnly start, DateOnly end)
    {
        if (end < start)
            throw new ArgumentException("End must be >= Start");
        (Start, End) = (start, end);
    }

    public int Days => End.DayNumber - Start.DayNumber;
    public bool Contains(DateOnly date) => date >= Start && date <= End;
    public bool Overlaps(DateRange other) => Start <= other.End && End >= other.Start;
}
```

## Enforcing Value Semantics with Interfaces

Define marker interfaces to enforce that value objects implement proper equality semantics. This makes the contract explicit and allows generic constraints.

```csharp
/// <summary>
/// Marker interface for value objects that have structural equality.
/// Implementers must provide value-based Equals and GetHashCode.
/// </summary>
public interface IValueObject : IEquatable<IValueObject>
{
}

/// <summary>
/// Strongly-typed value object interface.
/// </summary>
public interface IValueObject<T> : IEquatable<T> where T : IValueObject<T>
{
}

/// <summary>
/// Value object wrapping a single primitive value.
/// </summary>
public interface ISingleValueObject<T, TValue> : IValueObject<T>, IComparable<T>
    where T : ISingleValueObject<T, TValue>
    where TValue : IComparable<TValue>
{
    TValue Value { get; }
}
```

### Implementation Example

```csharp
public readonly record struct CustomerId(Guid Value) : ISingleValueObject<CustomerId, Guid>
{
    public int CompareTo(CustomerId other) => Value.CompareTo(other.Value);
    
    public static CustomerId New() => new(Guid.NewGuid());
    public static CustomerId FromGuid(Guid value) => new(value);
}

public readonly record struct Money(decimal Amount, Currency Currency) : IValueObject<Money>
{
    public static Money Zero(Currency currency) => new(0, currency);
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return new Money(Amount + other.Amount, Currency);
    }
}
```

### Benefits of Value Object Interfaces

- **Explicit contract**: Types that implement `IValueObject<T>` must provide proper equality
- **Generic constraints**: Write methods that work with any value object: `void Process<T>(T value) where T : IValueObject<T>`
- **Documentation**: Interface signals intent—this type has value semantics
- **Consistency**: All value objects in the codebase follow the same pattern

## See Also

- [Primitive Obsession](./primitive-obsession.md)
- [Data Clumps](./data-clump.md)
- [Static Factory Methods](./static-factory-methods.md)
