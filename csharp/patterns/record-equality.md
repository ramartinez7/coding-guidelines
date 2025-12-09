# Record Equality (Value-Based Equality)

> Use records for types where equality should be based on value, not reference identity—compiler-generated value semantics.

## Problem

Classes use reference equality by default, requiring manual implementation of `Equals()` and `GetHashCode()` for value-based comparison. Records provide value semantics automatically, reducing boilerplate and errors.

## Example

### ❌ Before

```csharp
public class Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    // Manual equality implementation - easy to get wrong
    public override bool Equals(object? obj)
    {
        if (obj is not Money other)
            return false;
        
        return Amount == other.Amount && Currency == other.Currency;
    }

    public override int GetHashCode()
    {
        return HashCode.Combine(Amount, Currency);
    }

    // Forgot to implement IEquatable<T> - performance issue
    // Forgot to override == and != operators
}

// Reference equality by default
var m1 = new Money(10.00m, "USD");
var m2 = new Money(10.00m, "USD");

Console.WriteLine(m1 == m2);        // False - different references!
Console.WriteLine(m1.Equals(m2));   // True - but easy to forget .Equals()
```

### ✅ After

```csharp
public enum CurrencyCode { USD, EUR, GBP, JPY }

public readonly record struct Money(decimal Amount, CurrencyCode Currency)
{
    // That's it! Compiler generates:
    // - Value-based Equals()
    // - GetHashCode()
    // - == and != operators
    // - IEquatable<Money>
    // - ToString()
    // - Deconstruction
    // - With-expressions for copies
}

// Value equality automatically
var m1 = new Money(10.00m, CurrencyCode.USD);
var m2 = new Money(10.00m, CurrencyCode.USD);

Console.WriteLine(m1 == m2);        // True - value equality!
Console.WriteLine(m1.Equals(m2));   // True

// Useful features for free
Console.WriteLine(m1);              // Money { Amount = 10.00, Currency = USD }

var (amount, currency) = m1;        // Deconstruction

var m3 = m1 with { Amount = 20.00m }; // Copy with modification
```

## Why It's Better

1. **Less boilerplate**: No manual equality implementation needed.

2. **Correct by default**: All equality members consistent and correct.

3. **Immutable**: Records encourage immutability with `init` properties.

4. **Value semantics**: Records are compared by value, not reference.

5. **Additional features**: ToString, deconstruction, with-expressions included.

## Record Types

### 1. Record Class (Reference Type)

Default record type—reference type with value semantics:

```csharp
public record Person(string FirstName, string LastName, int Age);

// Equivalent to:
public record Person
{
    public string FirstName { get; init; }
    public string LastName { get; init; }
    public int Age { get; init; }

    public Person(string firstName, string lastName, int age)
    {
        FirstName = firstName;
        LastName = lastName;
        Age = age;
    }
}

// Use for domain value objects
var person1 = new Person("Alice", "Smith", 30);
var person2 = new Person("Alice", "Smith", 30);

Console.WriteLine(person1 == person2);  // True - value equality
Console.WriteLine(ReferenceEquals(person1, person2)); // False - different objects
```

### 2. Record Struct (Value Type)

Lightweight value type with value semantics:

```csharp
public readonly record struct Point(int X, int Y);

// Best for small, simple types
public readonly record struct EmailAddress(string Value);

public readonly record struct DateRange(DateOnly Start, DateOnly End);

// Stack-allocated, no heap allocation
var point1 = new Point(10, 20);
var point2 = new Point(10, 20);

Console.WriteLine(point1 == point2);  // True
```

### 3. Positional Records vs Property Records

```csharp
// Positional syntax (concise)
public record Product(ProductId Id, string Name, Money Price);

// Property syntax (explicit)
public record Product
{
    public ProductId Id { get; init; }
    public string Name { get; init; }
    public Money Price { get; init; }
}

// Mixed (positional + additional members)
public record Order(OrderId Id, CustomerId CustomerId)
{
    public List<OrderItem> Items { get; init; } = new();
    
    public Money CalculateTotal()
    {
        return Items.Select(i => i.Price * i.Quantity).Sum();
    }
}
```

## Advanced Features

### With-Expressions (Non-Destructive Mutation)

Create modified copies without changing the original:

```csharp
public record Address(string Street, string City, string State, string Zip);

var address1 = new Address("123 Main St", "Austin", "TX", "78701");

// Create a copy with one property changed
var address2 = address1 with { City = "Dallas" };

Console.WriteLine(address1.City);  // Austin (unchanged)
Console.WriteLine(address2.City);  // Dallas
```

### Inheritance

Records support inheritance (record classes only, not record structs):

```csharp
public abstract record Payment(Money Amount);

public sealed record CashPayment(Money Amount) : Payment(Amount);

public sealed record CreditCardPayment(
    Money Amount,
    CardNumber CardNumber) : Payment(Amount);

public sealed record BankTransfer(
    Money Amount,
    AccountNumber AccountNumber) : Payment(Amount);

// Value equality respects hierarchy
Payment p1 = new CashPayment(Money.USD(100));
Payment p2 = new CashPayment(Money.USD(100));
Payment p3 = new CreditCardPayment(Money.USD(100), cardNumber);

Console.WriteLine(p1 == p2);  // True - same type and value
Console.WriteLine(p1 == p3);  // False - different types
```

### Custom Equality

Override equality logic when needed:

```csharp
public record Temperature(decimal Value, string Unit)
{
    // Custom equality that compares in Celsius
    public virtual bool Equals(Temperature? other)
    {
        if (other is null)
            return false;
        
        return ToCelsius(this) == ToCelsius(other);
    }

    public override int GetHashCode()
    {
        return ToCelsius(this).GetHashCode();
    }

    private static decimal ToCelsius(Temperature temp)
    {
        return temp.Unit switch
        {
            "C" => temp.Value,
            "F" => (temp.Value - 32) * 5 / 9,
            "K" => temp.Value - 273.15m,
            _ => throw new ArgumentException($"Unknown unit: {temp.Unit}")
        };
    }
}

var t1 = new Temperature(100, "C");
var t2 = new Temperature(212, "F");

Console.WriteLine(t1 == t2);  // True - both represent boiling point of water
```

## When to Use Records

Use records for:

✅ **Value Objects** in DDD—objects defined by their value, not identity:
```csharp
public record EmailAddress(string Value);
public record Money(decimal Amount, string Currency);
public record Address(string Street, string City, string State, string Zip);
```

✅ **DTOs** (Data Transfer Objects)—simple data carriers:
```csharp
public record CreateUserRequest(string Email, string Name);
public record UserResponse(Guid Id, string Email, string Name, DateTime CreatedAt);
```

✅ **Configuration objects**—read-only settings:
```csharp
public record DatabaseConfig(string ConnectionString, int MaxPoolSize, TimeSpan Timeout);
```

✅ **Immutable data structures**:
```csharp
public record OrderSnapshot(OrderId Id, CustomerId CustomerId, Money Total, DateTime CreatedAt);
```

## When Not to Use Records

Avoid records for:

❌ **Entities** with identity—use classes:
```csharp
// ❌ Bad - entity compared by value
public record Order(OrderId Id, CustomerId CustomerId);

// ✅ Good - entity compared by ID
public sealed class Order
{
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    
    public override bool Equals(object? obj)
    {
        return obj is Order other && Id == other.Id;
    }
    
    public override int GetHashCode() => Id.GetHashCode();
}
```

❌ **Mutable objects** that need to change state:
```csharp
// ❌ Bad - fighting against immutability
public record ShoppingCart
{
    public List<CartItem> Items { get; set; } = new();  // Mutable!
}

// ✅ Good - embrace immutability or use a class
public sealed class ShoppingCart
{
    private readonly List<CartItem> items = new();
    
    public void AddItem(CartItem item) => items.Add(item);
    public IReadOnlyList<CartItem> Items => items;
}
```

❌ **Types with complex equality logic** that doesn't align with structural equality:
```csharp
// Complex custom equality may be clearer as a class
```

## Record Structs for Performance

Use `record struct` for small, frequently allocated types:

```csharp
// Hot path value types
public readonly record struct Quantity(int Value);

public readonly record struct Coordinate(double Latitude, double Longitude);

public readonly record struct Result(bool Success, string? Error);

// Avoid boxing with struct records
public readonly record struct ValidationResult
{
    public bool IsValid { get; init; }
    public IReadOnlyList<string> Errors { get; init; }
}
```

## Migration Strategy

Converting a class to a record:

```csharp
// Before
public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
    public string State { get; set; }
    public string Zip { get; set; }
    
    // 50 lines of equality boilerplate...
}

// After
public record Address(string Street, string City, string State, string Zip);
```

If the class is already immutable with correct equality, the conversion is safe. Test equality behavior thoroughly after migrating.

## See Also

- [Value Semantics](./value-semantics.md) — when to use value-based equality
- [Entities vs Value Objects](./entities-vs-value-objects.md) — identity vs value equality
- [Primitive Obsession](./primitive-obsession.md) — creating value types
- [Immutable Collections](./immutable-collections.md) — fully immutable data structures
- [Snapshot Immutability](./snapshot-immutability.md) — true immutability vs readonly
