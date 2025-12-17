# Testing Object Graphs

> Test complex object graphs and deep equality—verify nested objects, circular references, and structural equivalence.

## Problem

Complex domain models contain nested objects, collections, and relationships. Testing equality of deep object graphs with standard assertions is verbose and error-prone.

## Pattern

Use FluentAssertions' `BeEquivalentTo()` to compare object graphs structurally, with customizable comparison rules for different scenarios.

## Example

### ❌ Before - Manual Deep Comparison

```csharp
[Fact]
public void MapOrder_ComplexOrder_MapsAllProperties()
{
    // Arrange
    var order = CreateComplexOrder();
    var mapper = new OrderMapper();

    // Act
    var dto = mapper.Map(order);

    // Assert - Tedious manual comparison
    Assert.Equal(order.Id, dto.Id);
    Assert.Equal(order.CustomerId, dto.CustomerId);
    Assert.Equal(order.Status.ToString(), dto.Status);
    Assert.Equal(order.Items.Count, dto.Items.Count);
    
    for (int i = 0; i < order.Items.Count; i++)
    {
        Assert.Equal(order.Items[i].ProductId, dto.Items[i].ProductId);
        Assert.Equal(order.Items[i].Quantity, dto.Items[i].Quantity);
        Assert.Equal(order.Items[i].Price.Amount, dto.Items[i].Price);
        Assert.Equal(order.Items[i].Price.Currency, dto.Items[i].Currency);
    }
    
    Assert.Equal(order.ShippingAddress.Street, dto.ShippingAddress.Street);
    Assert.Equal(order.ShippingAddress.City, dto.ShippingAddress.City);
    // ... many more comparisons
}
```

### ✅ After - Structural Comparison

```csharp
[Fact]
public void MapOrder_ComplexOrder_MapsAllProperties()
{
    // Arrange
    var order = CreateComplexOrder();
    var mapper = new OrderMapper();

    // Act
    var dto = mapper.Map(order);

    // Assert - Deep structural comparison
    dto.Should().BeEquivalentTo(order, options => options
        .ExcludingMissingMembers()
        .WithStrictOrdering());
}
```

## Basic Object Graph Comparison

### Simple Nested Objects

```csharp
public record Address(string Street, string City, string ZipCode);
public record Customer(string Name, string Email, Address Address);

[Fact]
public void CloneCustomer_DeepCopy_EquivalentToOriginal()
{
    // Arrange
    var original = new Customer(
        "John Doe",
        "john@example.com",
        new Address("123 Main St", "Boston", "02101"));

    // Act
    var clone = original with { }; // Record copy

    // Assert
    clone.Should().BeEquivalentTo(original);
}
```

### Complex Nested Hierarchies

```csharp
[Fact]
public void DeserializeOrder_ComplexJson_ReconstructsObjectGraph()
{
    // Arrange
    var original = Order.Create(
        CustomerId.New(),
        new[]
        {
            OrderLineItem.Create("PROD-1", 2, Money.USD(10m)),
            OrderLineItem.Create("PROD-2", 1, Money.USD(15m))
        },
        shippingAddress: new Address("123 Main St", "Boston", "02101"),
        billingAddress: new Address("456 Oak Ave", "Cambridge", "02139"));

    var json = JsonSerializer.Serialize(original);

    // Act
    var deserialized = JsonSerializer.Deserialize<Order>(json);

    // Assert
    deserialized.Should().BeEquivalentTo(original);
}
```

## Customizing Comparisons

### Excluding Properties

```csharp
[Fact]
public void UpdateOrder_ModifiesOnlyChangedProperties()
{
    // Arrange
    var original = CreateOrder();
    var updated = CreateOrder();
    updated.UpdateShippingAddress(new Address("New Street", "New City", "12345"));

    // Assert - Exclude changed properties, compare the rest
    updated.Should().BeEquivalentTo(original, options => options
        .Excluding(o => o.ShippingAddress)
        .Excluding(o => o.UpdatedAt)
        .Excluding(o => o.Version));
}
```

### Excluding Nested Properties

```csharp
[Fact]
public void MapToDto_IgnoresInternalState()
{
    // Arrange
    var order = CreateOrder();
    var mapper = new OrderMapper();

    // Act
    var dto = mapper.Map(order);

    // Assert
    dto.Should().BeEquivalentTo(order, options => options
        .ExcludingMissingMembers()
        .Excluding(o => o.DomainEvents)  // Exclude domain events
        .Excluding(o => o.Path("Items[*].InternalId")));  // Exclude nested property
}
```

### Including Only Specific Properties

```csharp
[Fact]
public void CopyOrder_CopiesOnlyBusinessData()
{
    // Arrange
    var source = CreateOrder();

    // Act
    var copy = CopyBusinessDataOnly(source);

    // Assert
    copy.Should().BeEquivalentTo(source, options => options
        .Including(o => o.CustomerId)
        .Including(o => o.Items)
        .Including(o => o.ShippingAddress));
}
```

## Collection Comparison

### Ordered Collections

```csharp
[Fact]
public void SortOrders_ByDate_MaintainsOrder()
{
    // Arrange
    var orders = CreateUnsortedOrders();

    // Act
    var sorted = orders.OrderBy(o => o.CreatedAt).ToList();

    // Assert
    sorted.Should().BeEquivalentTo(orders.OrderBy(o => o.CreatedAt), 
        options => options.WithStrictOrdering());
}
```

### Unordered Collections

```csharp
[Fact]
public void GetOrderItems_ReturnsAllItems_AnyOrder()
{
    // Arrange
    var order = CreateOrderWithMultipleItems();

    // Act
    var items = order.GetItems();

    // Assert - Order doesn't matter
    items.Should().BeEquivalentTo(order.Items, 
        options => options.WithoutStrictOrdering());
}
```

### Collection with Member Matching

```csharp
[Fact]
public void MergeOrderLists_CombinesCorrectly()
{
    // Arrange
    var list1 = CreateOrders(1, 2, 3);
    var list2 = CreateOrders(2, 3, 4);

    // Act
    var merged = MergeOrders(list1, list2);

    // Assert
    merged.Should().BeEquivalentTo(CreateOrders(1, 2, 3, 4), options => options
        .Using<Order>(ctx => ctx.Subject.Id.Should().Be(ctx.Expectation.Id))
        .WhenTypeIs<Order>());
}
```

## Type-Specific Comparisons

### DateTime Comparisons

```csharp
[Fact]
public void CreateOrder_SetsTimestamp_NearCurrentTime()
{
    // Arrange
    var customerId = CustomerId.New();
    var items = CreateOrderItems();

    // Act
    var order = Order.Create(customerId, items);

    // Assert
    var expected = new
    {
        CustomerId = customerId,
        CreatedAt = DateTime.UtcNow
    };

    order.Should().BeEquivalentTo(expected, options => options
        .Using<DateTime>(ctx => 
            ctx.Subject.Should().BeCloseTo(ctx.Expectation, TimeSpan.FromSeconds(1)))
        .WhenTypeIs<DateTime>());
}
```

### Money/Decimal Comparisons

```csharp
[Fact]
public void CalculateTotal_WithRounding_ApproximatelyCorrect()
{
    // Arrange
    var items = CreateItemsWithComplexPricing();
    var calculator = new TotalCalculator();

    // Act
    var result = calculator.Calculate(items);

    // Assert
    var expected = new { Total = 123.456m };
    
    result.Should().BeEquivalentTo(expected, options => options
        .Using<decimal>(ctx => 
            ctx.Subject.Should().BeApproximately(ctx.Expectation, 0.01m))
        .WhenTypeIs<decimal>());
}
```

## Circular References

### Handling Parent-Child Relationships

```csharp
public class Order
{
    public List<OrderLineItem> Items { get; init; } = new();
    
    public void AddItem(OrderLineItem item)
    {
        item.Order = this;  // Circular reference
        Items.Add(item);
    }
}

public class OrderLineItem
{
    public Order? Order { get; set; }  // Back-reference to parent
    public string ProductId { get; init; } = "";
    public int Quantity { get; init; }
}

[Fact]
public void AddItem_EstablishesParentChildRelationship()
{
    // Arrange
    var order = Order.Create(CustomerId.New());
    var item = OrderLineItem.Create("PROD-1", 2, Money.USD(10m));

    // Act
    order.AddItem(item);

    // Assert - Handles circular reference
    order.Should().BeEquivalentTo(new
    {
        Items = new[]
        {
            new { ProductId = "PROD-1", Quantity = 2 }
        }
    }, options => options
        .ExcludingMissingMembers()
        .IgnoringCyclicReferences());
}
```

## Runtime Type Comparison

### Polymorphic Collections

```csharp
public abstract record PaymentMethod;
public record CreditCard(string Number, string Expiry) : PaymentMethod;
public record BankTransfer(string AccountNumber, string RoutingNumber) : PaymentMethod;

[Fact]
public void LoadPaymentMethods_MixedTypes_PreservesPolymorphism()
{
    // Arrange
    var expected = new PaymentMethod[]
    {
        new CreditCard("1234-5678-9012-3456", "12/25"),
        new BankTransfer("98765432", "012345678")
    };

    var json = JsonSerializer.Serialize(expected);

    // Act
    var deserialized = JsonSerializer.Deserialize<PaymentMethod[]>(json);

    // Assert
    deserialized.Should().BeEquivalentTo(expected, options => options
        .RespectingRuntimeTypes());
}
```

## Custom Comparison Logic

### Value Object Comparison

```csharp
[Fact]
public void CompareOrders_WithValueObjectSemantics()
{
    // Arrange
    var order1 = CreateOrder();
    var order2 = CreateOrder();

    // Act & Assert
    order2.Should().BeEquivalentTo(order1, options => options
        .Using<Money>(ctx =>
        {
            ctx.Subject.Amount.Should().Be(ctx.Expectation.Amount);
            ctx.Subject.Currency.Should().Be(ctx.Expectation.Currency);
        })
        .WhenTypeIs<Money>()
        .Using<CustomerId>(ctx =>
        {
            ctx.Subject.Value.Should().Be(ctx.Expectation.Value);
        })
        .WhenTypeIs<CustomerId>());
}
```

## Assertion Modes

### Strict Structural Equality

```csharp
[Fact]
public void SerializeDeserialize_ExactCopy_StrictEquality()
{
    // Arrange
    var original = CreateComplexOrder();
    var json = JsonSerializer.Serialize(original);

    // Act
    var deserialized = JsonSerializer.Deserialize<Order>(json);

    // Assert - All properties must match exactly
    deserialized.Should().BeEquivalentTo(original, options => options
        .WithStrictOrdering()
        .ComparingByMembers<Order>());
}
```

### Loose Comparison

```csharp
[Fact]
public void MapToViewModel_FlexibleMapping()
{
    // Arrange
    var order = CreateOrder();
    var mapper = new ViewModelMapper();

    // Act
    var viewModel = mapper.Map(order);

    // Assert - Only check properties that exist in both
    viewModel.Should().BeEquivalentTo(order, options => options
        .ExcludingMissingMembers()
        .WithoutStrictOrdering()
        .ComparingByValue<Money>());
}
```

## Nested Member Selection

### Path-Based Property Selection

```csharp
[Fact]
public void UpdateAddress_OnlyAddressChanged()
{
    // Arrange
    var original = CreateOrder();
    var updated = original with { };
    updated.UpdateShippingAddress(new Address("New St", "New City", "12345"));

    // Assert - Compare everything except shipping address
    updated.Should().BeEquivalentTo(original, options => options
        .Excluding(ctx => ctx.Path == "ShippingAddress")
        .Excluding(ctx => ctx.Path.EndsWith("UpdatedAt")));
}
```

### Property Path Patterns

```csharp
[Fact]
public void CloneOrderWithoutIds_AllDataExceptIdentifiers()
{
    // Arrange
    var original = CreateOrder();

    // Act
    var clone = CloneWithoutIds(original);

    // Assert
    clone.Should().BeEquivalentTo(original, options => options
        .Excluding(ctx => ctx.Path.EndsWith("Id"))
        .Excluding(ctx => ctx.Path.Contains("CreatedAt"))
        .Excluding(ctx => ctx.Path.Contains("UpdatedAt")));
}
```

## Guidelines

1. **Use BeEquivalentTo for structure**: Deep comparison of object graphs
2. **Exclude changing properties**: Timestamps, IDs when appropriate
3. **Handle circular references**: Use `IgnoringCyclicReferences()`
4. **Customize comparisons**: Use type-specific comparisons for special types
5. **Test deserialization**: Verify round-trip serialization preserves structure

## Benefits

1. **Less code**: Avoid manual property-by-property comparison
2. **Maintainable**: Add properties without updating tests
3. **Clear intent**: Express structural equality directly
4. **Better errors**: See all differences when graphs don't match
5. **Flexible**: Customize comparison logic for domain needs

## Common Pitfalls

### Over-Excluding Properties

```csharp
// ❌ Too broad
options.ExcludingMissingMembers()  // Might hide real differences

// ✅ More specific
options
    .Excluding(o => o.InternalState)
    .Excluding(o => o.CachedValues)
```

### Ignoring Important Differences

```csharp
// ❌ Might miss bugs
options.WithoutStrictOrdering()  // When order matters

// ✅ Explicit about ordering requirements
options.WithStrictOrdering()  // For ordered collections
```

## See Also

- [Testing Value Objects](./testing-value-objects.md) — value equality testing
- [Testing Record Types](./testing-record-types.md) — record equality
- [Testing Collection Assertions](./testing-collection-assertions.md) — collection comparison
- [Snapshot Testing](./snapshot-testing.md) — baseline comparison
