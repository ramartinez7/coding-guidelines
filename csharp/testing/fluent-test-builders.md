# Fluent Test Builders

> Combine builders with fluent interfaces—expressive, maintainable test data setup.

## Problem

Test data builders can become verbose with many `With*` method calls. Fluent APIs make builders more expressive and readable by enabling semantic method names and chaining.

## Pattern

Enhance standard builders with:
- **Semantic methods** that capture domain concepts
- **Chainable API** for expressive configuration
- **Composite operations** that set multiple properties

## Example

### ❌ Before - Repetitive Builder Calls

```csharp
var order = OrderBuilder.Default()
    .WithCustomerId(customerId)
    .WithSubtotal(Money.USD(1000m))
    .WithStatus(OrderStatus.Pending)
    .WithCustomerTier(CustomerTier.Premium)
    .Build();
```

### ✅ After - Fluent Builder

```csharp
var order = OrderBuilder.Default()
    .ForCustomer(customerId)
    .WithHighValue()
    .AsPremium()
    .Build();
```

## Implementation

### Basic Fluent Methods

```csharp
public sealed class OrderBuilder
{
    private OrderId id = OrderId.New();
    private CustomerId customerId = CustomerId.New();
    private Money subtotal = Money.USD(100m);
    private OrderStatus status = OrderStatus.Pending;
    private CustomerTier tier = CustomerTier.Standard;

    private OrderBuilder() { }

    public static OrderBuilder Default() => new OrderBuilder();

    // Semantic methods
    public OrderBuilder ForCustomer(CustomerId value)
    {
        this.customerId = value;
        return this;
    }

    public OrderBuilder WithHighValue()
    {
        this.subtotal = Money.USD(1000m);
        return this;
    }

    public OrderBuilder WithLowValue()
    {
        this.subtotal = Money.USD(10m);
        return this;
    }

    public OrderBuilder AsPremium()
    {
        this.tier = CustomerTier.Premium;
        return this;
    }

    public OrderBuilder AsStandard()
    {
        this.tier = CustomerTier.Standard;
        return this;
    }

    public OrderBuilder InPendingState()
    {
        this.status = OrderStatus.Pending;
        return this;
    }

    public OrderBuilder InProcessedState()
    {
        this.status = OrderStatus.Processed;
        return this;
    }

    public Order Build()
    {
        return new Order(id, customerId, subtotal, status, tier);
    }
}
```

### Composite Operations

Methods that set multiple related properties:

```csharp
public sealed class OrderBuilder
{
    // ... fields

    public OrderBuilder AsNewOrder()
    {
        this.status = OrderStatus.Pending;
        this.createdAt = DateTime.UtcNow;
        this.subtotal = Money.USD(100m);
        return this;
    }

    public OrderBuilder AsShippedOrder()
    {
        this.status = OrderStatus.Shipped;
        this.shippedAt = DateTime.UtcNow;
        this.trackingNumber = TrackingNumber.Create("TRACK123");
        return this;
    }

    public OrderBuilder AsCompletedOrder()
    {
        this.status = OrderStatus.Completed;
        this.completedAt = DateTime.UtcNow;
        this.shippedAt = DateTime.UtcNow.AddDays(-3);
        return this;
    }

    public OrderBuilder WithRushShipping()
    {
        this.shippingMethod = ShippingMethod.Rush;
        this.shippingCost = Money.USD(25m);
        this.estimatedDelivery = DateTime.UtcNow.AddDays(1);
        return this;
    }
}

// Usage
var order = OrderBuilder.Default()
    .AsShippedOrder()
    .WithRushShipping()
    .Build();
```

### Expressive Time Periods

```csharp
public sealed class CustomerBuilder
{
    private DateTime registeredDate = DateTime.UtcNow;

    public CustomerBuilder RegisteredYearsAgo(int years)
    {
        this.registeredDate = DateTime.UtcNow.AddYears(-years);
        return this;
    }

    public CustomerBuilder RegisteredMonthsAgo(int months)
    {
        this.registeredDate = DateTime.UtcNow.AddMonths(-months);
        return this;
    }

    public CustomerBuilder RegisteredDaysAgo(int days)
    {
        this.registeredDate = DateTime.UtcNow.AddDays(-days);
        return this;
    }

    public CustomerBuilder RegisteredToday()
    {
        this.registeredDate = DateTime.UtcNow;
        return this;
    }
}

// Usage
var customer = CustomerBuilder.Default()
    .RegisteredYearsAgo(2)
    .Build();
```

### Collection Builders

```csharp
public sealed class OrderBuilder
{
    private List<OrderItem> items = new();

    public OrderBuilder WithItem(ProductId productId, int quantity)
    {
        this.items.Add(new OrderItem(productId, quantity));
        return this;
    }

    public OrderBuilder WithItems(params OrderItem[] items)
    {
        this.items.AddRange(items);
        return this;
    }

    public OrderBuilder WithThreeStandardItems()
    {
        return WithItem(ProductId.New(), quantity: 1)
            .WithItem(ProductId.New(), quantity: 2)
            .WithItem(ProductId.New(), quantity: 1);
    }

    public OrderBuilder WithEmptyCart()
    {
        this.items.Clear();
        return this;
    }
}

// Usage
var order = OrderBuilder.Default()
    .WithThreeStandardItems()
    .Build();
```

### Conditional Building

```csharp
public sealed class ProductBuilder
{
    private int stockQuantity = 100;
    private bool isDiscounted = false;
    private Money price = Money.USD(10m);

    public ProductBuilder InStock(int quantity = 100)
    {
        this.stockQuantity = quantity;
        return this;
    }

    public ProductBuilder OutOfStock()
    {
        this.stockQuantity = 0;
        return this;
    }

    public ProductBuilder OnSale(decimal discountPercentage = 0.20m)
    {
        this.isDiscounted = true;
        this.price = Money.USD(price.Amount * (1 - discountPercentage));
        return this;
    }

    public ProductBuilder AtPrice(Money value)
    {
        this.price = value;
        return this;
    }
}

// Usage
var product = ProductBuilder.Default()
    .OutOfStock()
    .OnSale()
    .Build();
```

## Extension Methods Pattern

Create extension methods to add domain-specific builders:

```csharp
public static class OrderBuilderExtensions
{
    public static OrderBuilder ForPremiumCustomer(this OrderBuilder builder)
    {
        return builder
            .AsPremium()
            .WithHighValue();
    }

    public static OrderBuilder ReadyToShip(this OrderBuilder builder)
    {
        return builder
            .InProcessedState()
            .WithValidShippingAddress();
    }

    public static OrderBuilder Expedited(this OrderBuilder builder)
    {
        return builder
            .WithRushShipping()
            .WithPriority();
    }
}

// Usage
var order = OrderBuilder.Default()
    .ForPremiumCustomer()
    .ReadyToShip()
    .Expedited()
    .Build();
```

## Testing Edge Cases

```csharp
public sealed class EmailAddressBuilder
{
    private string localPart = "test";
    private string domain = "example.com";

    public EmailAddressBuilder WithLongLocalPart()
    {
        this.localPart = new string('a', 64);
        return this;
    }

    public EmailAddressBuilder WithSpecialCharacters()
    {
        this.localPart = "test+tag";
        return this;
    }

    public EmailAddressBuilder WithSubdomain()
    {
        this.domain = "mail.example.com";
        return this;
    }

    public EmailAddressBuilder WithInternationalDomain()
    {
        this.domain = "例え.jp";
        return this;
    }

    public string Build() => $"{localPart}@{domain}";
}

// Usage
var email = EmailAddressBuilder.Default()
    .WithLongLocalPart()
    .WithSubdomain()
    .Build();
```

## Complete Example

```csharp
public sealed class ShoppingCartBuilder
{
    private CustomerId customerId = CustomerId.New();
    private List<CartItem> items = new();
    private Money subtotal = Money.USD(0m);
    private DiscountCode? discountCode = null;
    private DateTime createdAt = DateTime.UtcNow;

    private ShoppingCartBuilder() { }

    public static ShoppingCartBuilder Default() => new ShoppingCartBuilder();

    public ShoppingCartBuilder ForCustomer(CustomerId value)
    {
        this.customerId = value;
        return this;
    }

    public ShoppingCartBuilder AddProduct(ProductId productId, int quantity = 1)
    {
        this.items.Add(new CartItem(productId, quantity));
        RecalculateSubtotal();
        return this;
    }

    public ShoppingCartBuilder WithMultipleItems()
    {
        return AddProduct(ProductId.New(), quantity: 2)
            .AddProduct(ProductId.New(), quantity: 1)
            .AddProduct(ProductId.New(), quantity: 3);
    }

    public ShoppingCartBuilder WithDiscount(DiscountCode code)
    {
        this.discountCode = code;
        return this;
    }

    public ShoppingCartBuilder WithTwentyPercentOff()
    {
        this.discountCode = DiscountCode.Create("SAVE20", percentage: 0.20m);
        return this;
    }

    public ShoppingCartBuilder CreatedHoursAgo(int hours)
    {
        this.createdAt = DateTime.UtcNow.AddHours(-hours);
        return this;
    }

    public ShoppingCartBuilder Abandoned()
    {
        return CreatedHoursAgo(48);
    }

    public ShoppingCartBuilder Empty()
    {
        this.items.Clear();
        this.subtotal = Money.USD(0m);
        return this;
    }

    private void RecalculateSubtotal()
    {
        // Simplified calculation
        this.subtotal = Money.USD(items.Count * 10m);
    }

    public ShoppingCart Build()
    {
        return new ShoppingCart(customerId, items, subtotal, discountCode, createdAt);
    }
}

// Usage in tests
[Fact]
public void ApplyDiscount_ToCartWithMultipleItems_ReducesTotal()
{
    // Arrange
    var cart = ShoppingCartBuilder.Default()
        .WithMultipleItems()
        .WithTwentyPercentOff()
        .Build();

    // Act & Assert
    Assert.Equal(Money.USD(24m), cart.Total); // 30 * 0.80
}

[Fact]
public void SendReminderEmail_ForAbandonedCart_SendsEmail()
{
    // Arrange
    var cart = ShoppingCartBuilder.Default()
        .Abandoned()
        .WithMultipleItems()
        .Build();

    // Act
    var result = emailService.SendAbandonedCartReminder(cart);

    // Assert
    Assert.True(result.IsSuccess);
}
```

## Benefits

1. **Expressiveness**: Code reads like business specifications
2. **Discoverability**: Semantic methods reveal available options
3. **Maintainability**: Domain logic centralized in builder
4. **Reusability**: Common scenarios encapsulated in methods
5. **Flexibility**: Combine primitives and composites freely

## Guidelines

- Use domain language in method names
- Create composite methods for common scenarios
- Keep primitives (`WithId`, `WithDate`) for flexibility
- Add extension methods for cross-cutting concerns
- Don't make builders do too much—focus on configuration

## See Also

- [Test Data Builders](./test-data-builders.md) — basic builder pattern
- [Object Mother Pattern](./object-mother-pattern.md) — named standard fixtures
- [Arrange-Act-Assert Pattern](./arrange-act-assert.md) — using builders in tests
- [Testing Domain Invariants](./testing-domain-invariants.md) — builders for validation tests
