# Assertion Scopes

> Use FluentAssertions assertion scopes to report all failures at once—see all assertion failures in a single test run.

## Problem

When a test has multiple assertions, it stops at the first failure. You fix that issue, re-run the test, and discover the next failure. This cycle repeats, wasting time.

## Pattern

Use `AssertionScope` to collect all assertion failures and report them together. Fix all issues in one iteration instead of playing whack-a-mole.

## Example

### ❌ Before - Sequential Failures

```csharp
[Fact]
public void CreateOrder_ValidInput_ProducesCorrectOrder()
{
    // Arrange
    var customerId = CustomerId.New();
    var items = CreateOrderItems();

    // Act
    var order = Order.Create(customerId, items);

    // Assert
    order.CustomerId.Should().Be(customerId);     // Fails here
    order.Status.Should().Be(OrderStatus.Pending); // Never reached
    order.Items.Should().HaveCount(3);             // Never reached
    order.Total.Should().Be(Money.USD(100m));      // Never reached
}
```

**Output:**
```
Expected order.CustomerId to be Guid "abc...", but found Guid "xyz...".
```

Fix the customer ID issue, re-run, discover status issue, repeat.

### ✅ After - All Failures at Once

```csharp
[Fact]
public void CreateOrder_ValidInput_ProducesCorrectOrder()
{
    // Arrange
    var customerId = CustomerId.New();
    var items = CreateOrderItems();

    // Act
    var order = Order.Create(customerId, items);

    // Assert
    using (new AssertionScope())
    {
        order.CustomerId.Should().Be(customerId);
        order.Status.Should().Be(OrderStatus.Pending);
        order.Items.Should().HaveCount(3);
        order.Total.Should().Be(Money.USD(100m));
    }
}
```

**Output:**
```
Expected order.CustomerId to be Guid "abc...", but found Guid "xyz...".
Expected order.Status to be Pending, but found Draft.
Expected order.Items to contain 3 item(s), but found 2.
Expected order.Total to be USD 100.00, but found USD 75.00.
```

See all four failures immediately.

## Basic Usage

### Simple Assertion Scope

```csharp
[Fact]
public void Customer_AfterUpdate_HasCorrectProperties()
{
    // Arrange
    var customer = Customer.Create("john@example.com", "John Doe");

    // Act
    customer.UpdateEmail("jane@example.com");
    customer.UpdateName("Jane Doe");

    // Assert
    using (new AssertionScope())
    {
        customer.Email.Should().Be("jane@example.com");
        customer.Name.Should().Be("Jane Doe");
        customer.UpdatedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(1));
        customer.Version.Should().Be(2);
    }
}
```

### Named Assertion Scope

```csharp
[Fact]
public void Order_ComplexValidation_AllFieldsCorrect()
{
    // Arrange & Act
    var order = CreateComplexOrder();

    // Assert
    using (new AssertionScope("Order validation"))
    {
        order.Id.Should().NotBeEmpty();
        order.CustomerId.Should().NotBeEmpty();
        order.Status.Should().Be(OrderStatus.Pending);
        order.Items.Should().NotBeEmpty();
    }
}
```

**Output includes scope name:**
```
Order validation: Expected order.Id to not be empty, but found Guid.Empty.
```

## Complex Object Validation

### Validating Nested Objects

```csharp
[Fact]
public void ShoppingCart_AfterAddingItems_HasCorrectState()
{
    // Arrange
    var cart = ShoppingCart.Create(CustomerId.New());

    // Act
    cart.AddItem(CreateProduct("PROD-1", 10m), quantity: 2);
    cart.AddItem(CreateProduct("PROD-2", 15m), quantity: 1);

    // Assert
    using (new AssertionScope())
    {
        cart.Items.Should().HaveCount(2);
        cart.Total.Should().Be(Money.USD(35m));
        
        var firstItem = cart.Items.First();
        using (new AssertionScope("First item"))
        {
            firstItem.ProductId.Should().Be("PROD-1");
            firstItem.Quantity.Should().Be(2);
            firstItem.Subtotal.Should().Be(Money.USD(20m));
        }
        
        var secondItem = cart.Items.Last();
        using (new AssertionScope("Second item"))
        {
            secondItem.ProductId.Should().Be("PROD-2");
            secondItem.Quantity.Should().Be(1);
            secondItem.Subtotal.Should().Be(Money.USD(15m));
        }
    }
}
```

### Collection Element Validation

```csharp
[Fact]
public void ProcessOrders_MultipleOrders_AllProcessedCorrectly()
{
    // Arrange
    var orders = CreateTestOrders(3);
    var processor = new OrderProcessor();

    // Act
    var results = orders.Select(o => processor.Process(o)).ToList();

    // Assert
    using (new AssertionScope())
    {
        results.Should().HaveCount(3);
        results.Should().AllSatisfy(r => r.IsSuccess.Should().BeTrue());
        
        for (int i = 0; i < orders.Count; i++)
        {
            using (new AssertionScope($"Order {i}"))
            {
                orders[i].Status.Should().Be(OrderStatus.Processed);
                orders[i].ProcessedAt.Should().NotBeNull();
                orders[i].ProcessedAt.Should().BeAfter(orders[i].CreatedAt);
            }
        }
    }
}
```

## Testing Multiple Scenarios

### Parameterized Tests with Scopes

```csharp
[Theory]
[InlineData("john@example.com", "John Doe", true)]
[InlineData("invalid", "Jane Doe", false)]
[InlineData("test@test.com", "", false)]
public void CreateCustomer_VariousInputs_ValidatesCorrectly(
    string email, 
    string name, 
    bool shouldSucceed)
{
    // Act
    var result = Customer.Create(email, name);

    // Assert
    using (new AssertionScope())
    {
        result.IsSuccess.Should().Be(shouldSucceed);
        
        if (shouldSucceed)
        {
            result.Value.Email.Should().Be(email);
            result.Value.Name.Should().Be(name);
            result.Value.CreatedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(1));
        }
        else
        {
            result.Error.Should().NotBeNullOrWhiteSpace();
        }
    }
}
```

## Domain Invariant Testing

### Aggregate Root Validation

```csharp
[Fact]
public void Order_AfterShipment_MaintainsInvariants()
{
    // Arrange
    var order = CreateProcessedOrder();

    // Act
    order.Ship(CreateShippingInfo());

    // Assert - Check all invariants hold
    using (new AssertionScope("Order invariants after shipment"))
    {
        // Status invariants
        order.Status.Should().Be(OrderStatus.Shipped);
        order.ShippedAt.Should().NotBeNull();
        order.ShippedAt.Should().BeAfter(order.ProcessedAt);
        
        // Item invariants
        order.Items.Should().NotBeEmpty();
        order.Items.Should().AllSatisfy(item => 
        {
            using (new AssertionScope($"Item {item.ProductId}"))
            {
                item.Quantity.Should().BePositive();
                item.Price.Amount.Should().BePositive();
                item.Subtotal.Should().Be(item.Price.Multiply(item.Quantity));
            }
        });
        
        // Total invariant
        var expectedTotal = order.Items.Sum(i => i.Subtotal.Amount);
        order.Total.Amount.Should().Be(expectedTotal);
        
        // Shipping invariants
        order.ShippingAddress.Should().NotBeNull();
        order.ShippingMethod.Should().NotBeNull();
    }
}
```

## Testing State Transitions

### State Machine Validation

```csharp
[Fact]
public void Order_ThroughFullLifecycle_TransitionsCorrectly()
{
    // Arrange
    var order = Order.Create(CustomerId.New(), CreateOrderItems());

    // Assert initial state
    using (new AssertionScope("Initial state"))
    {
        order.Status.Should().Be(OrderStatus.Pending);
        order.ProcessedAt.Should().BeNull();
        order.ShippedAt.Should().BeNull();
        order.DeliveredAt.Should().BeNull();
    }

    // Act - Process
    order.Process();

    // Assert processed state
    using (new AssertionScope("After processing"))
    {
        order.Status.Should().Be(OrderStatus.Processed);
        order.ProcessedAt.Should().NotBeNull();
        order.ShippedAt.Should().BeNull();
        order.DeliveredAt.Should().BeNull();
    }

    // Act - Ship
    order.Ship(CreateShippingInfo());

    // Assert shipped state
    using (new AssertionScope("After shipping"))
    {
        order.Status.Should().Be(OrderStatus.Shipped);
        order.ProcessedAt.Should().NotBeNull();
        order.ShippedAt.Should().NotBeNull();
        order.DeliveredAt.Should().BeNull();
        order.ShippedAt.Should().BeAfter(order.ProcessedAt);
    }

    // Act - Deliver
    order.MarkAsDelivered();

    // Assert delivered state
    using (new AssertionScope("After delivery"))
    {
        order.Status.Should().Be(OrderStatus.Delivered);
        order.ProcessedAt.Should().NotBeNull();
        order.ShippedAt.Should().NotBeNull();
        order.DeliveredAt.Should().NotBeNull();
        order.DeliveredAt.Should().BeAfter(order.ShippedAt);
    }
}
```

## Custom Assertion Scope Extensions

### Helper Methods

```csharp
public static class AssertionScopeExtensions
{
    public static void AssertOrder(this Order order, Action<Order> assertions)
    {
        using (new AssertionScope($"Order {order.Id}"))
        {
            assertions(order);
        }
    }

    public static void AssertAll<T>(
        this IEnumerable<T> items, 
        Action<T, int> assertions)
    {
        var list = items.ToList();
        
        for (int i = 0; i < list.Count; i++)
        {
            using (new AssertionScope($"Item {i}"))
            {
                assertions(list[i], i);
            }
        }
    }
}
```

### Using Custom Extensions

```csharp
[Fact]
public void ProcessMultipleOrders_AllValid_ProcessedCorrectly()
{
    // Arrange
    var orders = CreateTestOrders(5);
    var processor = new OrderProcessor();

    // Act
    foreach (var order in orders)
    {
        processor.Process(order);
    }

    // Assert
    orders.AssertAll((order, index) =>
    {
        order.Status.Should().Be(OrderStatus.Processed);
        order.ProcessedAt.Should().NotBeNull();
    });
}
```

## Assertion Scope with BeEquivalentTo

### Comparing Complex Objects

```csharp
[Fact]
public void MapToDto_ComplexAggregate_MapsAllProperties()
{
    // Arrange
    var order = CreateComplexOrder();
    var mapper = new OrderMapper();

    // Act
    var dto = mapper.MapToDto(order);

    // Assert
    using (new AssertionScope())
    {
        dto.Should().BeEquivalentTo(order, options => options
            .ExcludingMissingMembers()
            .Using<DateTime>(ctx => ctx.Subject.Should().BeCloseTo(ctx.Expectation, TimeSpan.FromSeconds(1)))
            .WhenTypeIs<DateTime>());
    }
}
```

## Performance Considerations

### Lazy Evaluation

```csharp
[Fact]
public void ExpensiveValidation_MultipleChecks_AllEvaluated()
{
    // Arrange
    var data = LoadLargeDataset();

    // Assert - All checks run even if early ones fail
    using (new AssertionScope())
    {
        data.Count.Should().Be(1000);  // Fails
        
        // These still execute (if they're not dependent on Count)
        data.Should().AllSatisfy(item => 
            item.IsValid.Should().BeTrue());  // Still runs
            
        data.Sum(x => x.Value).Should().Be(50000);  // Still runs
    }
}
```

## Guidelines

1. **Use for related assertions**: Group logically related checks
2. **Name scopes**: Provide context for nested validations
3. **Don't overuse**: Not every test needs a scope
4. **Balance granularity**: Too many assertions in one scope can be overwhelming
5. **Nested scopes**: Use for hierarchical validation

## When to Use

**Use AssertionScope when:**
- Testing multiple properties of a single object
- Validating domain invariants
- Testing state transitions
- Comparing complex object graphs
- Validating collections of objects

**Don't use when:**
- Testing a single concern (one assertion)
- Assertions are independent scenarios
- Failures are cascading (later assertions depend on earlier ones)

## Benefits

1. **Faster debugging**: See all failures at once
2. **Complete picture**: Understand full extent of problems
3. **Time savings**: Fix all issues in one iteration
4. **Better test design**: Forces thinking about related assertions
5. **Clear test intent**: Grouped assertions show what's being validated

## Common Patterns

### Validate and Assert Pattern

```csharp
[Fact]
public void ComplexOperation_ProducesValidOutput()
{
    // Arrange & Act
    var result = PerformComplexOperation();

    // Assert - Validate output structure
    using (new AssertionScope("Output structure"))
    {
        result.Should().NotBeNull();
        result.Data.Should().NotBeNull();
        result.Metadata.Should().NotBeNull();
    }

    // Assert - Validate data correctness
    using (new AssertionScope("Data correctness"))
    {
        result.Data.Count.Should().BeGreaterThan(0);
        result.Data.Should().AllSatisfy(d => d.IsValid.Should().BeTrue());
    }

    // Assert - Validate metadata
    using (new AssertionScope("Metadata"))
    {
        result.Metadata.ProcessedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(1));
        result.Metadata.ProcessedBy.Should().NotBeNullOrWhiteSpace();
    }
}
```

## See Also

- [One Assertion Per Test](./one-assertion-per-test.md) — balancing focused tests with comprehensive validation
- [Testing Domain Invariants](./testing-domain-invariants.md) — validating business rules
- [Testing Collection Assertions](./testing-collection-assertions.md) — collection validation
- [Custom Assertions](./testing-custom-assertions.md) — domain-specific assertions
