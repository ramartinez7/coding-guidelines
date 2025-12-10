# Newtype Pattern (Zero-Cost Wrappers)

> Using primitives directly when you need nominal typing—use the newtype pattern to create compile-time distinct types with zero runtime overhead.

## Problem

You want the type safety of wrapper types (like strongly-typed IDs or branded primitives) but without the runtime cost of allocations, indirection, and boxing. Traditional wrapper classes add overhead, while type aliases (`using`) provide no type safety.

## Example

### ❌ Before

```csharp
// Type alias: no type safety
using CustomerId = System.Guid;
using OrderId = System.Guid;

public class OrderService
{
    // Can accidentally swap IDs—compiler won't catch it
    public void ProcessOrder(Guid customerId, Guid orderId)
    {
        // Wrong! But compiles fine
        var order = _orderRepo.GetById(customerId);
        var customer = _customerRepo.GetById(orderId);
    }
}

// Or: Wrapper class with runtime overhead
public class CustomerId
{
    public Guid Value { get; }
    public CustomerId(Guid value) => Value = value;
}

// Every wrapper instance allocates on heap
var ids = Enumerable.Range(0, 1000000)
    .Select(_ => new CustomerId(Guid.NewGuid()))  // 1M heap allocations
    .ToList();
```

**Problems:**
- Type aliases don't prevent mixing types
- Wrapper classes have allocation overhead
- Boxing/unboxing costs for struct wrappers
- GC pressure from allocations

### ✅ After (Newtype Pattern)

```csharp
/// <summary>
/// Zero-cost wrapper for Guid that creates a distinct compile-time type.
/// </summary>
public readonly record struct CustomerId
{
    readonly Guid value;
    
    CustomerId(Guid value)
    {
        this.value = value;
    }
    
    public static CustomerId NewId() => new(Guid.NewGuid());
    
    public static CustomerId From(Guid value) => new(value);
    
    // Explicit conversion prevents accidental mixing
    public Guid ToGuid() => value;
    
    // Equality based on underlying value
    public bool Equals(CustomerId other) => value.Equals(other.value);
    
    public override int GetHashCode() => value.GetHashCode();
    
    public override string ToString() => value.ToString();
}

public readonly record struct OrderId
{
    readonly Guid value;
    
    OrderId(Guid value)
    {
        this.value = value;
    }
    
    public static OrderId NewId() => new(Guid.NewGuid());
    
    public static OrderId From(Guid value) => new(value);
    
    public Guid ToGuid() => value;
}

public class OrderService
{
    // Type system prevents ID confusion
    public void ProcessOrder(CustomerId customerId, OrderId orderId)
    {
        var customer = _customerRepo.GetById(customerId);
        var order = _orderRepo.GetById(orderId);
        
        // ❌ Won't compile: cannot convert CustomerId to OrderId
        // var wrong = _orderRepo.GetById(customerId);
    }
}

// Zero heap allocations—struct is stack-allocated or inline in arrays
var ids = Enumerable.Range(0, 1000000)
    .Select(_ => CustomerId.NewId())  // No heap allocations
    .ToList();
```

## Newtype for Validated Values

```csharp
/// <summary>
/// Email address that's guaranteed to be valid.
/// Private constructor enforces validation at creation.
/// </summary>
public readonly record struct Email
{
    readonly string value;
    
    Email(string value)
    {
        this.value = value;
    }
    
    public static Result<Email, ValidationError> Create(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return Result<Email, ValidationError>.Failure(
                new ValidationError("Email cannot be empty"));
        
        if (!input.Contains('@'))
            return Result<Email, ValidationError>.Failure(
                new ValidationError("Email must contain @"));
        
        if (input.Length > 254)
            return Result<Email, ValidationError>.Failure(
                new ValidationError("Email too long"));
        
        return Result<Email, ValidationError>.Success(new Email(input));
    }
    
    // Safe to convert to string—guaranteed valid
    public override string ToString() => value;
    
    // Implicit conversion is safe (always valid)
    public static implicit operator string(Email email) => email.value;
}

// Usage: Parse, don't validate
var emailResult = Email.Create(input);

if (!emailResult.IsSuccess)
    return BadRequest(emailResult.Error);

var email = emailResult.Value;
SendWelcomeEmail(email);  // Type proves it's valid

void SendWelcomeEmail(Email email)
{
    // No validation needed—type guarantees it
    _emailService.Send(email.ToString(), "Welcome!", body);
}
```

## Newtype for Units of Measure

```csharp
public readonly record struct Meters
{
    readonly double value;
    
    Meters(double value)
    {
        this.value = value;
    }
    
    public static Meters FromMeters(double value) => new(value);
    
    public double ToMeters() => value;
    
    public Feet ToFeet() => Feet.FromFeet(value * 3.28084);
    
    public static Meters operator +(Meters a, Meters b) =>
        new(a.value + b.value);
    
    public static Meters operator -(Meters a, Meters b) =>
        new(a.value - b.value);
    
    public static Meters operator *(Meters m, double scalar) =>
        new(m.value * scalar);
}

public readonly record struct Feet
{
    readonly double value;
    
    Feet(double value)
    {
        this.value = value;
    }
    
    public static Feet FromFeet(double value) => new(value);
    
    public double ToFeet() => value;
    
    public Meters ToMeters() => Meters.FromMeters(value / 3.28084);
    
    public static Feet operator +(Feet a, Feet b) =>
        new(a.value + b.value);
}

// Usage: Type system prevents dimensional errors
var height = Meters.FromMeters(10.5);
var width = Meters.FromMeters(5.2);
var area = height * width.ToMeters();  // Type-safe multiplication

var distanceInFeet = Feet.FromFeet(100);

// ❌ Won't compile: cannot add Meters and Feet
// var wrong = height + distanceInFeet;

// ✅ Must convert explicitly
var total = height + distanceInFeet.ToMeters();
```

## Newtype for Currency

```csharp
public readonly record struct Money
{
    readonly decimal amount;
    readonly string currencyCode;
    
    Money(decimal amount, string currencyCode)
    {
        this.amount = amount;
        this.currencyCode = currencyCode;
    }
    
    public static Money USD(decimal amount) => new(amount, "USD");
    
    public static Money EUR(decimal amount) => new(amount, "EUR");
    
    public decimal Amount => amount;
    
    public string CurrencyCode => currencyCode;
    
    public static Money operator +(Money a, Money b)
    {
        if (a.currencyCode != b.currencyCode)
            throw new InvalidOperationException(
                $"Cannot add {a.currencyCode} and {b.currencyCode}");
        
        return new Money(a.amount + b.amount, a.currencyCode);
    }
    
    public static Money operator *(Money money, decimal multiplier) =>
        new(money.amount * multiplier, money.currencyCode);
    
    public override string ToString() => $"{amount:N2} {currencyCode}";
}

// Usage: Prevents currency mixing
var priceInDollars = Money.USD(19.99m);
var taxInDollars = Money.USD(2.00m);
var totalInDollars = priceInDollars + taxInDollars;  // OK

var priceInEuros = Money.EUR(17.50m);

// ❌ Runtime error: cannot mix currencies
try
{
    var invalid = priceInDollars + priceInEuros;
}
catch (InvalidOperationException)
{
    // Must convert first
    var convertedPrice = ConvertToUSD(priceInEuros);
    var total = priceInDollars + convertedPrice;
}
```

## Generic Newtype Helper

```csharp
/// <summary>
/// Generic newtype pattern for any underlying type.
/// Inherit to create zero-cost wrappers.
/// </summary>
public abstract record struct Newtype<T> where T : notnull
{
    protected readonly T Value;
    
    protected Newtype(T value)
    {
        Value = value;
    }
    
    public override string ToString() => Value.ToString() ?? string.Empty;
}

// Usage: Create specific newtypes by inheritance
public readonly record struct ProductId : IEquatable<ProductId>
{
    readonly Guid value;
    
    ProductId(Guid value)
    {
        this.value = value;
    }
    
    public static ProductId NewId() => new(Guid.NewGuid());
    
    public static ProductId From(Guid value) => new(value);
    
    public bool Equals(ProductId other) => value.Equals(other.value);
    
    public override int GetHashCode() => value.GetHashCode();
}

public readonly record struct CategoryId : IEquatable<CategoryId>
{
    readonly int value;
    
    CategoryId(int value)
    {
        this.value = value;
    }
    
    public static CategoryId From(int value) => new(value);
    
    public int ToInt32() => value;
    
    public bool Equals(CategoryId other) => value.Equals(other.value);
    
    public override int GetHashCode() => value.GetHashCode();
}
```

## Serialization Support

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

public readonly record struct OrderId
{
    readonly Guid value;
    
    OrderId(Guid value)
    {
        this.value = value;
    }
    
    public static OrderId NewId() => new(Guid.NewGuid());
    
    public static OrderId From(Guid value) => new(value);
    
    public Guid ToGuid() => value;
}

/// <summary>
/// JSON converter for OrderId newtype.
/// Serializes as the underlying Guid.
/// </summary>
public class OrderIdJsonConverter : JsonConverter<OrderId>
{
    public override OrderId Read(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options)
    {
        var guid = reader.GetGuid();
        return OrderId.From(guid);
    }
    
    public override void Write(
        Utf8JsonWriter writer,
        OrderId value,
        JsonSerializerOptions options)
    {
        writer.WriteStringValue(value.ToGuid());
    }
}

// Register converter
var options = new JsonSerializerOptions
{
    Converters = { new OrderIdJsonConverter() }
};

// Serializes as: "orderId": "123e4567-e89b-12d3-a456-426614174000"
var json = JsonSerializer.Serialize(new { OrderId = OrderId.NewId() }, options);
```

## Entity Framework Core Support

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Microsoft.EntityFrameworkCore.Storage.ValueConversion;

public class Order
{
    public OrderId Id { get; set; }
    public CustomerId CustomerId { get; set; }
    // ... other properties
}

public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        // Convert OrderId newtype to Guid for database storage
        builder.Property(o => o.Id)
            .HasConversion(
                orderId => orderId.ToGuid(),
                guid => OrderId.From(guid));
        
        builder.Property(o => o.CustomerId)
            .HasConversion(
                customerId => customerId.ToGuid(),
                guid => CustomerId.From(guid));
    }
}
```

## Key Principles

### 1. Use readonly record struct

```csharp
// ✅ Preferred: Zero overhead, immutable
public readonly record struct UserId
{
    readonly Guid value;
    UserId(Guid value) => this.value = value;
}
```

### 2. Private Constructor

```csharp
// ✅ Force creation through named factory methods
UserId(Guid value) => this.value = value;

public static UserId NewId() => new(Guid.NewGuid());
public static UserId From(Guid value) => new(value);
```

### 3. Explicit Conversions

```csharp
// ✅ Require explicit conversion to underlying type
public Guid ToGuid() => value;

// ❌ Avoid implicit operators (can hide type safety)
// public static implicit operator Guid(UserId id) => id.value;
```

### 4. Implement Equality

```csharp
// record struct provides this automatically, but explicit is clearer
public bool Equals(UserId other) => value.Equals(other.value);
public override int GetHashCode() => value.GetHashCode();
```

## Why It's a Problem

1. **Runtime overhead**: Reference type wrappers allocate on heap
2. **GC pressure**: Million wrapped values = million allocations
3. **No type safety**: Type aliases don't prevent mixing
4. **Boxing costs**: Regular structs box when used as objects

## Symptoms

- Using type aliases that don't provide type safety
- Heap allocations for simple wrapper types
- Frequent boxing/unboxing in hot paths
- Passing primitives where domain types would be clearer

## Benefits

- **Zero runtime cost**: No allocations, no indirection, no boxing
- **Compile-time safety**: Cannot mix incompatible types
- **Performance**: Stack allocation, inline in arrays
- **Clarity**: Types document intent better than primitives

## Trade-offs

- **Boilerplate**: Each newtype requires definition
- **Conversion overhead**: Must explicitly convert to underlying type
- **Serialization**: Requires custom converters for JSON/EF
- **Generic constraints**: Some generic code may need adjustment

## See Also

- [Strongly Typed IDs](./strongly-typed-ids.md) — wrapper pattern for identifiers
- [Branded Primitives](./branded-primitives.md) — nominal typing over structural
- [Units of Measure](./units-of-measure.md) — dimensional analysis
- [Value Semantics](./value-semantics.md) — value-based equality
