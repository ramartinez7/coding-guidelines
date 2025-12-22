# Record Types Best Practices

> Use record types for value-based equality, immutability, and concise data models in C# 9+.

## Problem

Creating value objects with proper equality, immutability, and toString requires significant boilerplate code.

## Example

### ❌ Before

```csharp
public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
    public string State { get; set; }
    public string ZipCode { get; set; }

    public override bool Equals(object? obj)
    {
        if (obj is not Address other) return false;
        return Street == other.Street &&
               City == other.City &&
               State == other.State &&
               ZipCode == other.ZipCode;
    }

    public override int GetHashCode()
    {
        return HashCode.Combine(Street, City, State, ZipCode);
    }
}
```

### ✅ After

```csharp
public sealed record Address(
    string Street,
    string City,
    string State,
    string ZipCode);
// Equality, GetHashCode, ToString, Deconstruct auto-generated!
```

## Best Practices

### 1. Use Records for Value Objects

```csharp
// ✅ Records for value objects
public sealed record Money(decimal Amount, string Currency);
public sealed record EmailAddress(string Value);
public sealed record Coordinate(double Latitude, double Longitude);
public sealed record DateRange(DateTime Start, DateTime End);
```

### 2. Use Positional Records for Simple Types

```csharp
// ✅ Positional syntax for simple records
public sealed record Point(int X, int Y);

public sealed record OrderId(Guid Value);

public sealed record PersonName(string First, string Last);

// Usage with deconstruction
var (x, y) = new Point(10, 20);
var (firstName, lastName) = new PersonName("John", "Doe");
```

### 3. Use Property Syntax for Complex Types

```csharp
// ✅ Property syntax for records with logic
public sealed record Customer
{
    public required string Name { get; init; }
    public required EmailAddress Email { get; init; }
    public Address? BillingAddress { get; init; }
    public Address? ShippingAddress { get; init; }

    public Address EffectiveShippingAddress =>
        ShippingAddress ?? BillingAddress ?? throw new InvalidOperationException();
}
```

### 4. Use 'with' for Immutable Updates

```csharp
// ✅ Non-destructive mutation with 'with'
var originalOrder = new Order(
    Id: new OrderId(Guid.NewGuid()),
    Total: 100m,
    Status: OrderStatus.Pending);

var confirmedOrder = originalOrder with
{
    Status = OrderStatus.Confirmed
};

// Original unchanged
Assert.That(originalOrder.Status, Is.EqualTo(OrderStatus.Pending));
Assert.That(confirmedOrder.Status, Is.EqualTo(OrderStatus.Confirmed));
```

### 5. Use Init-Only Properties for Validation

```csharp
// ✅ Validation in init accessor
public sealed record EmailAddress
{
    private string value;

    public string Value
    {
        get => value;
        init
        {
            if (string.IsNullOrWhiteSpace(value))
            {
                throw new ArgumentException("Email cannot be empty");
            }

            if (!value.Contains("@"))
            {
                throw new ArgumentException("Invalid email format");
            }

            this.value = value;
        }
    }
}

// Usage
var email = new EmailAddress { Value = "test@example.com" };  // OK
var invalid = new EmailAddress { Value = "invalid" };  // Throws
```

### 6. Seal Records by Default

```csharp
// ✅ Sealed records (prevent inheritance issues)
public sealed record Money(decimal Amount, string Currency);

public sealed record UserId(Guid Value);

// Only use non-sealed for inheritance hierarchies
public abstract record PaymentMethod;
public sealed record CreditCard(string Last4) : PaymentMethod;
public sealed record PayPal(string Email) : PaymentMethod;
```

### 7. Use Record Structs for Performance

```csharp
// ✅ Record struct for value types
public readonly record struct Point(int X, int Y);

public readonly record struct DateOnly(int Year, int Month, int Day);

// Avoid boxing, stack-allocated
var point = new Point(10, 20);  // No heap allocation
```

### 8. Override Equality for Custom Comparisons

```csharp
// ✅ Custom equality in record
public sealed record CaseInsensitiveString(string Value)
{
    public virtual bool Equals(CaseInsensitiveString? other)
    {
        return other != null &&
            string.Equals(Value, other.Value, StringComparison.OrdinalIgnoreCase);
    }

    public override int GetHashCode()
    {
        return StringComparer.OrdinalIgnoreCase.GetHashCode(Value);
    }
}

var s1 = new CaseInsensitiveString("Hello");
var s2 = new CaseInsensitiveString("HELLO");
Assert.That(s1, Is.EqualTo(s2));  // True
```

### 9. Use Primary Constructor for Dependencies

```csharp
// ✅ Primary constructor with DI (C# 12+)
public sealed class OrderService(
    IOrderRepository repository,
    ILogger<OrderService> logger)
{
    public Result<Order, OrderError> GetOrder(OrderId id)
    {
        logger.LogInformation("Getting order {OrderId}", id);
        return repository.Find(id);
    }
}
```

### 10. Combine with Pattern Matching

```csharp
public abstract record PaymentMethod;
public sealed record CreditCard(string Last4, string Type) : PaymentMethod;
public sealed record PayPal(string Email) : PaymentMethod;
public sealed record BankTransfer(string AccountNumber) : PaymentMethod;

// ✅ Pattern matching on records
public string DescribePayment(PaymentMethod payment) => payment switch
{
    CreditCard { Type: "Visa" } cc => $"Visa ending in {cc.Last4}",
    CreditCard cc => $"Card ending in {cc.Last4}",
    PayPal pp => $"PayPal account {pp.Email}",
    BankTransfer bt => $"Bank transfer to {bt.AccountNumber}",
    _ => "Unknown payment method"
};
```

### 11. Use for DTOs and API Models

```csharp
// ✅ Records perfect for DTOs
public sealed record CreateOrderRequest(
    CustomerId CustomerId,
    List<OrderItemDto> Items,
    Address ShippingAddress);

public sealed record OrderItemDto(
    ProductId ProductId,
    int Quantity);

public sealed record OrderResponse(
    OrderId Id,
    decimal Total,
    OrderStatus Status,
    DateTime CreatedAt);
```

### 12. Don't Mix Records with Entities

```csharp
// ❌ Using record for entity (identity-based)
public record Order  // NO - Orders have identity
{
    public OrderId Id { get; init; }
    public List<OrderItem> Items { get; init; }
}

// ✅ Use class for entities
public class Order  // YES - Has identity
{
    public OrderId Id { get; }
    private List<OrderItem> items = new();

    public IReadOnlyList<OrderItem> Items => items.AsReadOnly();
}

// ✅ Use record for value objects
public sealed record OrderItem(  // YES - Defined by its values
    ProductId ProductId,
    int Quantity,
    decimal Price);
```

## Symptoms

- Verbose boilerplate for simple data classes
- Bugs from reference equality when value equality needed
- Mutable types causing unexpected bugs
- Repetitive equality implementations

## Benefits

- **Concise syntax** with auto-generated members
- **Value equality** by default
- **Immutability** with init-only properties
- **Pattern matching** support
- **Non-destructive mutation** with 'with' expression

## See Also

- [Value Semantics](./value-semantics.md) — Value-based equality
- [Record Equality](./record-equality.md) — Compiler-generated equality
- [Immutable Collections](./immutable-collections.md) — Immutable data structures
