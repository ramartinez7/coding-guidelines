# Object Mother Pattern

> Centralize creation of standard test fixtures—reusable, named configurations of test objects.

## Problem

Tests repeatedly create the same standard configurations of objects. The creation logic gets duplicated, and when requirements change, many tests need updating.

## Pattern

Create static factory methods that return commonly-used test objects with meaningful names. Unlike builders, Object Mothers create fully-configured objects for specific scenarios.

## Example

### ❌ Before - Duplicated Setup

```csharp
[Fact]
public void Test1()
{
    var customer = Customer.Create(
        CustomerId.New(),
        "John Doe",
        "john@example.com",
        CustomerTier.Premium,
        registeredDate: DateTime.UtcNow.AddYears(-2)
    );
}

[Fact]
public void Test2()
{
    var customer = Customer.Create(
        CustomerId.New(),
        "John Doe",
        "john@example.com",
        CustomerTier.Premium,
        registeredDate: DateTime.UtcNow.AddYears(-2)
    );
}

[Fact]
public void Test3()
{
    var customer = Customer.Create(
        CustomerId.New(),
        "Jane Smith",
        "jane@example.com",
        CustomerTier.Standard,
        registeredDate: DateTime.UtcNow.AddMonths(-1)
    );
}
```

### ✅ After - Object Mother

```csharp
public static class CustomerMother
{
    public static Customer PremiumCustomer() => Customer.Create(
        CustomerId.New(),
        "John Doe",
        "john@example.com",
        CustomerTier.Premium,
        registeredDate: DateTime.UtcNow.AddYears(-2)
    );

    public static Customer StandardCustomer() => Customer.Create(
        CustomerId.New(),
        "Jane Smith",
        "jane@example.com",
        CustomerTier.Standard,
        registeredDate: DateTime.UtcNow.AddMonths(-1)
    );

    public static Customer NewCustomer() => Customer.Create(
        CustomerId.New(),
        "Bob Jones",
        "bob@example.com",
        CustomerTier.Standard,
        registeredDate: DateTime.UtcNow
    );

    public static Customer VipCustomer() => Customer.Create(
        CustomerId.New(),
        "Alice Williams",
        "alice@example.com",
        CustomerTier.VIP,
        registeredDate: DateTime.UtcNow.AddYears(-5)
    );

    public static Customer InactiveCustomer() => Customer.Create(
        CustomerId.New(),
        "Inactive User",
        "inactive@example.com",
        CustomerTier.Standard,
        registeredDate: DateTime.UtcNow.AddYears(-3),
        lastActivity: DateTime.UtcNow.AddYears(-1)
    );
}

// Usage
[Fact]
public void Test1()
{
    var customer = CustomerMother.PremiumCustomer();
    // ...
}

[Fact]
public void Test2()
{
    var customer = CustomerMother.PremiumCustomer();
    // ...
}

[Fact]
public void Test3()
{
    var customer = CustomerMother.StandardCustomer();
    // ...
}
```

## Implementation Patterns

### Simple Object Mother

```csharp
public static class OrderMother
{
    public static Order DefaultOrder() => Order.Create(
        OrderId.New(),
        CustomerId.New(),
        Money.USD(100m)
    );

    public static Order HighValueOrder() => Order.Create(
        OrderId.New(),
        CustomerId.New(),
        Money.USD(10000m)
    );

    public static Order EmptyOrder() => Order.Create(
        OrderId.New(),
        CustomerId.New(),
        Money.USD(0m)
    );
}
```

### Object Mother with Parameters

Allow some customization while maintaining standard configurations:

```csharp
public static class OrderMother
{
    public static Order PremiumOrder(CustomerId? customerId = null) => Order.Create(
        OrderId.New(),
        customerId ?? CustomerId.New(),
        Money.USD(1000m),
        CustomerTier.Premium
    );

    public static Order StandardOrder(Money? total = null) => Order.Create(
        OrderId.New(),
        CustomerId.New(),
        total ?? Money.USD(100m),
        CustomerTier.Standard
    );
}

// Usage
var order1 = OrderMother.PremiumOrder();
var order2 = OrderMother.PremiumOrder(customerId: myCustomerId);
var order3 = OrderMother.StandardOrder(total: Money.USD(500m));
```

### Compositional Object Mothers

Mother objects can use other mothers:

```csharp
public static class OrderMother
{
    public static Order ForPremiumCustomer() => Order.Create(
        OrderId.New(),
        CustomerMother.PremiumCustomer().Id,
        Money.USD(1000m)
    );

    public static Order ForNewCustomer() => Order.Create(
        OrderId.New(),
        CustomerMother.NewCustomer().Id,
        Money.USD(50m)
    );
}
```

### Object Mother for Complex Scenarios

```csharp
public static class OrderMother
{
    public static Order ShippedOrder()
    {
        var order = Order.Create(
            OrderId.New(),
            CustomerId.New(),
            Money.USD(100m)
        );
        
        order.Process();
        order.Ship(TrackingNumber.Create("TRACK123"));
        
        return order;
    }

    public static Order CancelledOrder()
    {
        var order = Order.Create(
            OrderId.New(),
            CustomerId.New(),
            Money.USD(100m)
        );
        
        order.Cancel(CancellationReason.CustomerRequest);
        
        return order;
    }

    public static Order CompletedOrder()
    {
        var order = ShippedOrder();
        order.Complete();
        
        return order;
    }
}
```

## Object Mother vs Builder

**Use Object Mother when:**
- Standard configurations are commonly reused
- You want named, semantic fixture creation
- The object represents a well-known business concept

**Use Builder when:**
- Each test needs different combinations
- Flexibility is more important than named configurations
- You want to specify only relevant properties

**Use both together:**

```csharp
public static class OrderMother
{
    public static Order PremiumOrder() => OrderBuilder.Default()
        .WithCustomerTier(CustomerTier.Premium)
        .WithSubtotal(Money.USD(1000m))
        .Build();

    public static Order StandardOrder() => OrderBuilder.Default()
        .WithCustomerTier(CustomerTier.Standard)
        .WithSubtotal(Money.USD(100m))
        .Build();
}
```

## Guidelines

**Use descriptive names**
- Names should describe the business scenario, not technical details
- `PremiumCustomer()` not `CustomerWithTier2()`
- `ExpiredOrder()` not `OrderWithOldDate()`

**Keep them simple**
- Object Mothers should be easy to understand
- If creation logic is complex, factor it out
- Document any non-obvious behavior

**Don't make them too generic**
- Avoid `CreateCustomer(tier, date, email, ...)`—that's a builder
- Object Mothers represent specific, named scenarios

**Organize by domain concept**
- One Mother class per aggregate
- Group related scenarios together

**Version control is documentation**
- As tests evolve, Object Mothers capture team knowledge
- They document what configurations are meaningful

## Benefits

1. **DRY**: Eliminate duplicated setup code
2. **Semantic**: Names communicate business intent
3. **Maintainable**: Change fixtures in one place
4. **Discoverable**: Easy to find standard configurations
5. **Documentation**: Captures meaningful scenarios

## Complete Example

```csharp
public static class ProductMother
{
    public static Product StandardProduct() => Product.Create(
        ProductId.New(),
        "Widget",
        "A standard widget",
        Money.USD(10m),
        stockQuantity: 100
    );

    public static Product ExpensiveProduct() => Product.Create(
        ProductId.New(),
        "Premium Gadget",
        "An expensive premium item",
        Money.USD(1000m),
        stockQuantity: 10
    );

    public static Product OutOfStockProduct() => Product.Create(
        ProductId.New(),
        "Rare Item",
        "Currently unavailable",
        Money.USD(50m),
        stockQuantity: 0
    );

    public static Product DiscountedProduct()
    {
        var product = StandardProduct();
        product.ApplyDiscount(Percentage.FromDecimal(0.20m)); // 20% off
        return product;
    }
}

// Tests
[Fact]
public void CalculateTotal_WithStandardProduct_ReturnsCorrectAmount()
{
    var product = ProductMother.StandardProduct();
    var total = product.Price * 3;
    Assert.Equal(Money.USD(30m), total);
}

[Fact]
public void AddToCart_OutOfStockProduct_ReturnsError()
{
    var product = ProductMother.OutOfStockProduct();
    var result = cart.Add(product, quantity: 1);
    Assert.True(result.IsFailure);
}
```

## See Also

- [Test Data Builders](./test-data-builders.md) — flexible object creation
- [Fluent Test Builders](./fluent-test-builders.md) — combining builders with fluent APIs
- [Test Isolation and Independence](./test-isolation-independence.md) — creating fresh test data
- [Arrange-Act-Assert Pattern](./arrange-act-assert.md) — simplifying test structure
