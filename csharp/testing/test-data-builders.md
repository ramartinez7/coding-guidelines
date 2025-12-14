# Builder Pattern for Test Data

> Build complex test objects fluently—specify only what matters for each test, use sensible defaults for the rest.

## Problem

Creating complex domain objects in tests requires many parameters. Tests become cluttered with irrelevant setup, and changes to constructors break many tests even when the change is unrelated to what they're testing.

## Pattern

Use the Builder pattern to create test objects with:
- **Sensible defaults** for all properties
- **Fluent API** to override only what's relevant
- **Immutable result** that can't be accidentally modified

## Example

### ❌ Before - Constructor Overload

```csharp
[Fact]
public void ProcessOrder_WithHighValueOrder_AppliesDiscount()
{
    // Arrange - Most of this is irrelevant boilerplate
    var order = new Order(
        id: OrderId.New(),
        customerId: CustomerId.New(),
        customerName: "John Doe",
        customerEmail: "john@example.com",
        shippingAddress: new Address("123 Main", "Seattle", "WA", "98101"),
        billingAddress: new Address("123 Main", "Seattle", "WA", "98101"),
        items: new List<OrderItem>(),
        subtotal: Money.USD(1000m),  // ← This is what matters
        tax: Money.USD(0m),
        shipping: Money.USD(0m),
        discount: Money.USD(0m),
        total: Money.USD(1000m),
        status: OrderStatus.Pending,
        createdAt: DateTime.UtcNow,
        updatedAt: DateTime.UtcNow
    );

    // Act
    var result = discountService.Apply(order);

    // Assert
    Assert.True(result.Discount > Money.USD(0m));
}
```

### ✅ After - Test Data Builder

```csharp
[Fact]
public void ProcessOrder_WithHighValueOrder_AppliesDiscount()
{
    // Arrange - Only specify what matters
    var order = OrderBuilder.Default()
        .WithSubtotal(Money.USD(1000m))
        .Build();

    // Act
    var result = discountService.Apply(order);

    // Assert
    Assert.True(result.Discount > Money.USD(0m));
}
```

## Implementation

```csharp
public sealed class OrderBuilder
{
    private OrderId id = OrderId.New();
    private CustomerId customerId = CustomerId.New();
    private string customerName = "Test Customer";
    private string customerEmail = "test@example.com";
    private Address shippingAddress = Address.Default();
    private Address billingAddress = Address.Default();
    private List<OrderItem> items = new();
    private Money subtotal = Money.USD(100m);
    private Money tax = Money.USD(0m);
    private Money shipping = Money.USD(0m);
    private Money discount = Money.USD(0m);
    private OrderStatus status = OrderStatus.Pending;
    private DateTime createdAt = DateTime.UtcNow;

    private OrderBuilder() { }

    public static OrderBuilder Default() => new OrderBuilder();

    public OrderBuilder WithId(OrderId value)
    {
        this.id = value;
        return this;
    }

    public OrderBuilder WithCustomerId(CustomerId value)
    {
        this.customerId = value;
        return this;
    }

    public OrderBuilder WithCustomerName(string value)
    {
        this.customerName = value;
        return this;
    }

    public OrderBuilder WithSubtotal(Money value)
    {
        this.subtotal = value;
        return this;
    }

    public OrderBuilder WithStatus(OrderStatus value)
    {
        this.status = value;
        return this;
    }

    public OrderBuilder AddItem(OrderItem item)
    {
        this.items.Add(item);
        return this;
    }

    public Order Build()
    {
        var total = subtotal + tax + shipping - discount;
        
        return new Order(
            id,
            customerId,
            customerName,
            customerEmail,
            shippingAddress,
            billingAddress,
            items,
            subtotal,
            tax,
            shipping,
            discount,
            total,
            status,
            createdAt,
            createdAt
        );
    }
}
```

## Usage Examples

### Basic Usage

```csharp
var order = OrderBuilder.Default().Build();
```

### Override Specific Properties

```csharp
var order = OrderBuilder.Default()
    .WithCustomerName("Jane Doe")
    .WithSubtotal(Money.USD(500m))
    .Build();
```

### Complex Setup

```csharp
var customerId = CustomerId.New();
var productId = ProductId.New();

var order = OrderBuilder.Default()
    .WithCustomerId(customerId)
    .WithStatus(OrderStatus.Shipped)
    .AddItem(new OrderItem(productId, quantity: 3, price: Money.USD(25m)))
    .WithSubtotal(Money.USD(75m))
    .Build();
```

### Named Builder Methods

Create semantic helper methods for common scenarios:

```csharp
public static class OrderBuilderExtensions
{
    public static OrderBuilder ForPremiumCustomer(this OrderBuilder builder)
    {
        return builder
            .WithCustomerName("Premium Customer")
            .WithSubtotal(Money.USD(1000m));
    }

    public static OrderBuilder Shipped(this OrderBuilder builder)
    {
        return builder
            .WithStatus(OrderStatus.Shipped);
    }
}

// Usage
var order = OrderBuilder.Default()
    .ForPremiumCustomer()
    .Shipped()
    .Build();
```

## Builder for Value Objects

```csharp
public sealed class AddressBuilder
{
    private string street = "123 Test Street";
    private string city = "Test City";
    private string state = "TS";
    private string zipCode = "12345";

    private AddressBuilder() { }

    public static AddressBuilder Default() => new AddressBuilder();

    public AddressBuilder WithStreet(string value)
    {
        this.street = value;
        return this;
    }

    public AddressBuilder WithCity(string value)
    {
        this.city = value;
        return this;
    }

    public AddressBuilder WithState(string value)
    {
        this.state = value;
        return this;
    }

    public AddressBuilder WithZipCode(string value)
    {
        this.zipCode = value;
        return this;
    }

    public Address Build() => Address.Create(street, city, state, zipCode);
}
```

## Guidelines

**Provide sensible defaults**
- All properties should have reasonable default values
- Tests should work with `Builder.Default().Build()` unless testing edge cases

**Make it fluent**
- Return `this` from all `With*` methods
- Enable method chaining

**Keep it in test projects**
- Builders are test-only code
- Don't put them in production assemblies

**One builder per aggregate**
- Build the root entity and its owned entities together
- Don't create separate builders for entities that can't exist independently

**Use object initializers for simple cases**
- If the object is simple, builders may be overkill
- Only use builders when complexity justifies them

## Benefits

1. **Focus**: Tests specify only relevant data
2. **Resilience**: Constructor changes don't break unrelated tests
3. **Readability**: Clear what matters for each test
4. **Maintainability**: Change defaults in one place
5. **Discoverability**: IDE autocomplete shows available properties

## See Also

- [Fluent Test Builders](./fluent-test-builders.md) — advanced fluent API techniques
- [Object Mother Pattern](./object-mother-pattern.md) — named standard fixtures
- [Arrange-Act-Assert Pattern](./arrange-act-assert.md) — simplifying test structure
- [Test Isolation and Independence](./test-isolation-independence.md) — creating fresh test data
