# Strongly Typed IDs

> Multiple ID parameters of the same primitive type—wrap each in a distinct type to prevent swapping.

## Problem

When methods accept multiple IDs of the same underlying type (`Guid`, `int`, `string`), the compiler can't distinguish between them. Swapped arguments compile successfully but fail catastrophically at runtime.

## Example

### ❌ Before

```csharp
public class OrderService
{
    // All three are GUIDs—compiler treats them identically
    public void AddItem(Guid userId, Guid orderId, Guid productId) 
    {
        // ...
    }
}

// Caller
Guid uId = Guid.NewGuid();
Guid oId = Guid.NewGuid();
Guid pId = Guid.NewGuid();

// Compiles perfectly, fails catastrophically at runtime
svc.AddItem(oId, uId, pId);  // Wrong order! But no compiler error.
```

### ✅ After

```csharp
public readonly record struct UserId(Guid Value) : ISingleValueObject<UserId, Guid>
{
    public static UserId New() => new(Guid.NewGuid());
    public int CompareTo(UserId other) => Value.CompareTo(other.Value);
    public override string ToString() => Value.ToString();
}

public readonly record struct OrderId(Guid Value) : ISingleValueObject<OrderId, Guid>
{
    public static OrderId New() => new(Guid.NewGuid());
    public int CompareTo(OrderId other) => Value.CompareTo(other.Value);
    public override string ToString() => Value.ToString();
}

public readonly record struct ProductId(Guid Value) : ISingleValueObject<ProductId, Guid>
{
    public static ProductId New() => new(Guid.NewGuid());
    public int CompareTo(ProductId other) => Value.CompareTo(other.Value);
    public override string ToString() => Value.ToString();
}

public class OrderService
{
    // Each parameter is now a distinct type
    public void AddItem(UserId userId, OrderId orderId, ProductId productId) 
    {
        // ...
    }
}

// Caller
var uId = UserId.New();
var oId = OrderId.New();
var pId = ProductId.New();

// Compiler error: Cannot convert OrderId to UserId
svc.AddItem(oId, uId, pId);  // ❌ Won't compile!

// Correct usage
svc.AddItem(uId, oId, pId);  // ✅ Types enforce correct order
```

## Why It's a Problem

1. **Argument swapping**: The most common source of subtle bugs—everything compiles, tests might even pass with mock data, but production fails.

2. **Loss of intent**: A `Guid` is an implementation detail; a `CustomerId` is a domain concept with meaning.

3. **No IDE help**: Autocomplete suggests any `Guid` variable for any `Guid` parameter.

4. **Refactoring hazards**: Reordering parameters breaks callers silently.

## Symptoms

- Methods with multiple `Guid`, `int`, `string`, or `long` parameters
- Bugs discovered in production where "the wrong ID was passed"
- Comments like `// userId, not orderId!`
- Variable naming conventions to compensate (`customerIdGuid`, `orderIdGuid`)
- Unit tests that don't catch ID mix-ups because all test IDs are the same

## Reducing Boilerplate

### Option 1: Base Record with Generic

```csharp
// The "Curiously Recurring Template Pattern" (CRTP)
// T refers to the derived type itself, enabling type-safe equality and factory methods
public abstract record StronglyTypedId<T>(Guid Value) where T : StronglyTypedId<T>
{
    public override string ToString() => Value.ToString();
}

// UserId passes itself as T, so the base class "knows" the concrete type
public sealed record UserId(Guid Value) : StronglyTypedId<UserId>(Value)
{
    public static UserId New() => new(Guid.NewGuid());
}
```

**Why `where T : StronglyTypedId<T>`?**

This constraint is called the "Curiously Recurring Template Pattern" (CRTP). It ensures that:

1. **Type-safe returns**: If you add a method like `T WithNewValue(Guid v)` to the base, it returns the correct derived type, not the base.
2. **Correct equality**: `UserId` only equals other `UserId` instances, never `OrderId`, even though both inherit from `StronglyTypedId<T>`.
3. **Self-referencing**: The derived class "tells" the base class what it is, enabling generic operations that return the right type.

```csharp
// Without CRTP, you'd need casting everywhere:
StronglyTypedId Clone() => /* returns base type, caller must cast */

// With CRTP, you get the concrete type back:
T Clone() => /* returns UserId when called on UserId */
```

### Option 2: Source Generator

Use a source generator library like [StronglyTypedId](https://github.com/andrewlock/StronglyTypedId):

```csharp
[StronglyTypedId]
public partial struct UserId { }

[StronglyTypedId]
public partial struct OrderId { }

[StronglyTypedId(backingType: StronglyTypedIdBackingType.Int)]
public partial struct ProductId { }
```

### Option 3: Minimal Struct

For maximum performance with minimal ceremony:

```csharp
public readonly struct UserId : IEquatable<UserId>
{
    public Guid Value { get; }
    public UserId(Guid value) => Value = value;
    
    public static UserId New() => new(Guid.NewGuid());
    
    public bool Equals(UserId other) => Value == other.Value;
    public override bool Equals(object? obj) => obj is UserId other && Equals(other);
    public override int GetHashCode() => Value.GetHashCode();
    public override string ToString() => Value.ToString();
    
    public static bool operator ==(UserId left, UserId right) => left.Equals(right);
    public static bool operator !=(UserId left, UserId right) => !left.Equals(right);
}
```

## JSON Serialization

Configure serializers to handle strongly typed IDs:

```csharp
// System.Text.Json
public class StronglyTypedIdConverter<TId, TValue> : JsonConverter<TId>
    where TId : ISingleValueObject<TId, TValue>
{
    public override TId Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        var value = JsonSerializer.Deserialize<TValue>(ref reader, options);
        return (TId)Activator.CreateInstance(typeof(TId), value)!;
    }

    public override void Write(Utf8JsonWriter writer, TId value, JsonSerializerOptions options)
    {
        JsonSerializer.Serialize(writer, value.Value, options);
    }
}
```

## Entity Framework Core

```csharp
// Value converter for EF Core
public class UserIdConverter : ValueConverter<UserId, Guid>
{
    public UserIdConverter() : base(
        id => id.Value,
        guid => new UserId(guid))
    { }
}

// In DbContext
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Order>()
        .Property(o => o.UserId)
        .HasConversion<UserIdConverter>();
}
```

## Benefits

- **Compile-time safety**: Swapped arguments don't compile
- **Self-documenting**: `AddItem(UserId, OrderId, ProductId)` is unambiguous
- **Refactoring-safe**: Reordering parameters causes compiler errors at all call sites
- **IDE support**: Autocomplete only suggests variables of the correct type
- **Domain modeling**: IDs become first-class domain concepts

## See Also

- [Primitive Obsession](./primitive-obsession.md)
- [Value Semantics](./value-semantics.md)
- [Static Factory Methods](./static-factory-methods.md)
