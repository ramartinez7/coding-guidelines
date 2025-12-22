# Pattern Matching Best Practices

> Use modern C# pattern matching for clear, exhaustive, and type-safe conditional logic.

## Problem

Traditional if-else chains and switch statements with casts are verbose, error-prone, and don't guarantee exhaustive handling.

## Example

### ❌ Before

```csharp
public string ProcessPayment(object payment)
{
    if (payment is CreditCardPayment)
    {
        var cc = (CreditCardPayment)payment;
        return $"Processing credit card ending in {cc.Last4}";
    }
    else if (payment is PayPalPayment)
    {
        var pp = (PayPalPayment)payment;
        return $"Processing PayPal account {pp.Email}";
    }
    else if (payment is BankTransferPayment)
    {
        var bt = (BankTransferPayment)payment;
        return $"Processing transfer to {bt.AccountNumber}";
    }
    else
    {
        return "Unknown payment type";
    }
}
```

### ✅ After

```csharp
public string ProcessPayment(Payment payment) => payment switch
{
    CreditCardPayment cc => $"Processing credit card ending in {cc.Last4}",
    PayPalPayment pp => $"Processing PayPal account {pp.Email}",
    BankTransferPayment bt => $"Processing transfer to {bt.AccountNumber}",
    _ => throw new ArgumentException($"Unknown payment type: {payment.GetType()}")
};
```

## Best Practices

### 1. Use Switch Expressions

```csharp
// ❌ Traditional switch statement
string GetStatusMessage(OrderStatus status)
{
    switch (status)
    {
        case OrderStatus.Pending:
            return "Order is pending";
        case OrderStatus.Confirmed:
            return "Order is confirmed";
        case OrderStatus.Shipped:
            return "Order has shipped";
        case OrderStatus.Delivered:
            return "Order delivered";
        default:
            return "Unknown status";
    }
}

// ✅ Switch expression
string GetStatusMessage(OrderStatus status) => status switch
{
    OrderStatus.Pending => "Order is pending",
    OrderStatus.Confirmed => "Order is confirmed",
    OrderStatus.Shipped => "Order has shipped",
    OrderStatus.Delivered => "Order delivered",
    _ => throw new ArgumentException($"Unknown status: {status}")
};
```

### 2. Use Type Patterns

```csharp
// ✅ Type pattern with variable declaration
public decimal CalculateFee(Payment payment) => payment switch
{
    CreditCardPayment cc => cc.Amount * 0.029m,
    PayPalPayment pp => pp.Amount * 0.034m,
    BankTransferPayment => 0m,  // No variable needed
    _ => throw new ArgumentException("Unknown payment type")
};
```

### 3. Use Property Patterns

```csharp
// ✅ Property pattern matching
public string GetShippingMessage(Order order) => order switch
{
    { Status: OrderStatus.Pending } => "Awaiting payment",
    { Status: OrderStatus.Paid, Total: > 100 } => "Free shipping!",
    { Status: OrderStatus.Paid } => "Shipping cost: $5.99",
    { Status: OrderStatus.Shipped } => "On the way",
    { Status: OrderStatus.Delivered } => "Delivered",
    _ => "Unknown"
};
```

### 4. Use Positional Patterns with Records

```csharp
public sealed record Point(int X, int Y);

// ✅ Positional pattern
public string GetQuadrant(Point point) => point switch
{
    (0, 0) => "Origin",
    (> 0, > 0) => "Quadrant I",
    (< 0, > 0) => "Quadrant II",
    (< 0, < 0) => "Quadrant III",
    (> 0, < 0) => "Quadrant IV",
    (0, _) => "On X-axis",
    (_, 0) => "On Y-axis"
};
```

### 5. Use Relational Patterns

```csharp
// ✅ Relational patterns
public string GetPriceCategory(decimal price) => price switch
{
    < 10 => "Budget",
    >= 10 and < 50 => "Standard",
    >= 50 and < 100 => "Premium",
    >= 100 => "Luxury",
    _ => throw new ArgumentException("Invalid price")
};
```

### 6. Use Logical Patterns (and, or, not)

```csharp
// ✅ Logical patterns
public bool IsBusinessHours(DateTime time) => time.Hour switch
{
    >= 9 and < 17 => true,
    _ => false
};

public string GetDayType(DayOfWeek day) => day switch
{
    DayOfWeek.Saturday or DayOfWeek.Sunday => "Weekend",
    not (DayOfWeek.Saturday or DayOfWeek.Sunday) => "Weekday"
};
```

### 7. Use Null Patterns

```csharp
// ✅ Null pattern matching
public string GetCustomerName(Customer? customer) => customer switch
{
    null => "Guest",
    { Name: not null } => customer.Name,
    _ => "Unknown"
};

// ✅ Not-null pattern (C# 9+)
if (customer is not null)
{
    ProcessCustomer(customer);
}
```

### 8. Use Nested Patterns

```csharp
// ✅ Nested patterns
public string GetOrderDescription(Order order) => order switch
{
    { Status: OrderStatus.Shipped, ShippingAddress: { Country: "USA" } } 
        => "Domestic shipment",
    { Status: OrderStatus.Shipped, ShippingAddress: { Country: not "USA" } }
        => "International shipment",
    { Status: OrderStatus.Pending, Items.Count: > 10 }
        => "Large pending order",
    _ => "Regular order"
};
```

### 9. Use List Patterns (C# 11+)

```csharp
// ✅ List patterns
public string DescribeArray(int[] numbers) => numbers switch
{
    [] => "Empty",
    [var single] => $"Single element: {single}",
    [var first, var second] => $"Two elements: {first}, {second}",
    [var first, .., var last] => $"Starts with {first}, ends with {last}",
    _ => "Multiple elements"
};
```

### 10. Use Guards (when Clauses)

```csharp
// ✅ Guards for additional conditions
public decimal CalculateDiscount(Order order) => order switch
{
    { Total: > 100 } when order.Customer.IsPremium => order.Total * 0.20m,
    { Total: > 100 } => order.Total * 0.10m,
    { Total: > 50 } when order.Customer.IsPremium => order.Total * 0.10m,
    _ => 0m
};
```

### 11. Use Tuple Patterns

```csharp
// ✅ Tuple pattern for multiple conditions
public string GetRiskLevel(decimal amount, string country) => (amount, country) switch
{
    (> 10000, "HighRiskCountry") => "High Risk",
    (> 10000, _) => "Medium Risk",
    (> 1000, "HighRiskCountry") => "Medium Risk",
    _ => "Low Risk"
};
```

### 12. Ensure Exhaustiveness

```csharp
// ✅ Exhaustive matching (no default, compiler checks)
public abstract record PaymentMethod;
public sealed record CreditCard : PaymentMethod;
public sealed record PayPal : PaymentMethod;

public string ProcessPayment(PaymentMethod method) => method switch
{
    CreditCard cc => ProcessCreditCard(cc),
    PayPal pp => ProcessPayPal(pp)
    // Compiler warning if new payment method added without handling
};
```

## Symptoms

- Verbose if-else chains with type checks
- Runtime casts causing InvalidCastException
- Missing cases not caught at compile time
- Difficult to understand complex conditions
- Repeated type checking and casting

## Benefits

- **Concise code** with switch expressions
- **Type safety** without explicit casts
- **Exhaustive checking** with compiler help
- **Readable logic** with declarative patterns
- **Better refactoring** with compile-time checks

## See Also

- [Discriminated Unions](./discriminated-unions.md) — Sum types with pattern matching
- [Exhaustive Pattern Matching](./exhaustive-pattern-matching.md) — Compiler-enforced completeness
- [Typed Errors](./typed-errors.md) — Pattern matching on errors
