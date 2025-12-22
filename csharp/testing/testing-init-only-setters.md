# Testing Init-Only Setters

> Test init-only properties and immutability—verify properties can only be set during initialization.

## Problem

Init-only properties enforce immutability after construction. Tests must verify properties can be set during initialization but not afterwards.

## Example

```csharp
public class Order
{
    public Guid Id { get; init; }
    public CustomerId CustomerId { get; init; }
    public DateTime CreatedAt { get; init; }
    public List<OrderLineItem> Items { get; init; } = new();
}

[Fact]
public void InitOnlyProperty_DuringObjectInitialization_CanBeSet()
{
    var customerId = CustomerId.New();
    var createdAt = DateTime.UtcNow;
    
    var order = new Order
    {
        Id = Guid.NewGuid(),
        CustomerId = customerId,
        CreatedAt = createdAt
    };
    
    order.CustomerId.Should().Be(customerId);
    order.CreatedAt.Should().Be(createdAt);
}

[Fact]
public void InitOnlyProperty_AfterConstruction_CannotBeModified()
{
    var order = new Order
    {
        Id = Guid.NewGuid(),
        CustomerId = CustomerId.New(),
        CreatedAt = DateTime.UtcNow
    };
    
    // This would not compile:
    // order.CustomerId = CustomerId.New();
}
```

## Testing Required Init Properties

```csharp
public class Product
{
    public required string Name { get; init; }
    public required decimal Price { get; init; }
}

[Fact]
public void RequiredInit_MustBeSet_DuringConstruction()
{
    var product = new Product
    {
        Name = "Widget",
        Price = 9.99m
    };
    
    product.Name.Should().Be("Widget");
    product.Price.Should().Be(9.99m);
}
```

## See Also

- [Testing Record Types](./testing-record-types.md)
- [Testing Value Objects](./testing-value-objects.md)
- [Testing Readonly Collections](./testing-readonly-collections.md)
