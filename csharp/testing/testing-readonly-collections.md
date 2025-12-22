# Testing Readonly and Immutable Collections

> Test readonly and immutable collections—verify immutability guarantees, prevent modifications, and ensure thread safety.

## Problem

Mutable collections can be modified unexpectedly, breaking invariants. Tests must verify that readonly and immutable collections enforce their immutability contracts.

## Pattern

Test that readonly collections cannot be modified and that immutable collections return new instances on modification attempts.

## Example

### ❌ Before - Mutable Collection Leaks

```csharp
public class Order
{
    private List<OrderLineItem> items = new();
    
    // Exposes mutable collection
    public List<OrderLineItem> Items => items;
}

// Caller can modify
var order = new Order();
order.Items.Add(newItem);  // Bypasses validation!
```

### ✅ After - Readonly Collection

```csharp
public class Order
{
    private readonly List<OrderLineItem> items = new();
    
    // Readonly view
    public IReadOnlyList<OrderLineItem> Items => items.AsReadOnly();
    
    public void AddItem(OrderLineItem item)
    {
        // Validation happens here
        items.Add(item);
    }
}

[Fact]
public void Items_ReturnsReadonlyCollection_CannotBeModified()
{
    // Arrange
    var order = Order.Create(CustomerId.New());
    
    // Act
    var items = order.Items;
    
    // Assert
    items.Should().BeAssignableTo<IReadOnlyList<OrderLineItem>>();
    items.Should().NotBeAssignableTo<IList<OrderLineItem>>();
}
```

## Testing IReadOnlyCollection

### Read-Only Access

```csharp
[Fact]
public void GetItems_ReturnsReadOnlyCollection()
{
    // Arrange
    var order = CreateOrderWithItems(3);
    
    // Act
    var items = order.Items;
    
    // Assert
    items.Should().HaveCount(3);
    items.Should().BeAssignableTo<IReadOnlyCollection<OrderLineItem>>();
}

[Fact]
public void Items_CastToList_ThrowsInvalidCastException()
{
    // Arrange
    var order = CreateOrderWithItems(3);
    
    // Act
    Action act = () => { var list = (IList<OrderLineItem>)order.Items; };
    
    // Assert
    act.Should().Throw<InvalidCastException>();
}
```

## Testing ImmutableList

### Immutable Operations

```csharp
using System.Collections.Immutable;

public record OrderState
{
    public ImmutableList<OrderLineItem> Items { get; init; } = ImmutableList<OrderLineItem>.Empty;
    
    public OrderState AddItem(OrderLineItem item)
    {
        return this with { Items = Items.Add(item) };
    }
}

[Fact]
public void AddItem_ReturnsNewInstance_OriginalUnchanged()
{
    // Arrange
    var original = new OrderState();
    var item = OrderLineItem.Create("PROD-1", 1, Money.USD(10m));
    
    // Act
    var updated = original.AddItem(item);
    
    // Assert
    original.Items.Should().BeEmpty();
    updated.Items.Should().HaveCount(1);
    original.Should().NotBeSameAs(updated);
}

[Fact]
public void ImmutableList_Add_ReturnsNewList()
{
    // Arrange
    var original = ImmutableList.Create(1, 2, 3);
    
    // Act
    var updated = original.Add(4);
    
    // Assert
    original.Should().HaveCount(3);
    updated.Should().HaveCount(4);
    original.Should().NotContain(4);
    updated.Should().Contain(4);
}
```

## Testing FrozenSet and FrozenDictionary

### .NET 8+ Frozen Collections

```csharp
using System.Collections.Frozen;

public class ProductCatalog
{
    private readonly FrozenDictionary<string, Product> products;
    
    public ProductCatalog(IEnumerable<Product> items)
    {
        products = items.ToFrozenDictionary(p => p.Id);
    }
    
    public Product? GetProduct(string id) =>
        products.TryGetValue(id, out var product) ? product : null;
}

[Fact]
public void FrozenDictionary_CannotBeModified()
{
    // Arrange
    var catalog = new ProductCatalog(CreateProducts());
    
    // Act & Assert
    catalog.GetProduct("PROD-1").Should().NotBeNull();
    // Cannot add/remove from frozen collection
}
```

## Testing Collection Initialization

### Init-Only Collections

```csharp
public record CustomerData
{
    public required string Name { get; init; }
    public required IReadOnlyList<string> PhoneNumbers { get; init; }
}

[Fact]
public void PhoneNumbers_AfterInit_CannotBeModified()
{
    // Arrange
    var data = new CustomerData 
    { 
        Name = "John",
        PhoneNumbers = new List<string> { "123-456" }.AsReadOnly()
    };
    
    // Act & Assert
    data.PhoneNumbers.Should().BeAssignableTo<IReadOnlyList<string>>();
}
```

## Testing AsReadOnly()

### ReadOnlyCollection Wrapper

```csharp
[Fact]
public void AsReadOnly_CreatesReadonlyView_BackingListChangesReflected()
{
    // Arrange
    var backingList = new List<int> { 1, 2, 3 };
    var readonly = backingList.AsReadOnly();
    
    // Act
    backingList.Add(4);
    
    // Assert
    readonly.Should().HaveCount(4);
    readonly.Should().Contain(4);
}

[Fact]
public void AsReadOnly_CannotBeModified()
{
    // Arrange
    var list = new List<int> { 1, 2, 3 };
    var readOnly = list.AsReadOnly();
    
    // Act & Assert
    readOnly.Should().NotBeAssignableTo<IList<int>>();
}
```

## Testing Defensive Copies

### Returning Copies

```csharp
public class ShoppingCart
{
    private readonly List<CartItem> items = new();
    
    public IReadOnlyList<CartItem> GetItems()
    {
        return items.ToList().AsReadOnly();
    }
    
    public ImmutableList<CartItem> GetItemsImmutable()
    {
        return items.ToImmutableList();
    }
}

[Fact]
public void GetItems_ReturnsDefensiveCopy_InternalListNotExposed()
{
    // Arrange
    var cart = new ShoppingCart();
    var item1 = CreateCartItem();
    cart.AddItem(item1);
    
    // Act
    var items1 = cart.GetItems();
    cart.AddItem(CreateCartItem());
    var items2 = cart.GetItems();
    
    // Assert
    items1.Should().HaveCount(1);
    items2.Should().HaveCount(2);
}
```

## Testing Thread Safety

### Concurrent Reads

```csharp
[Fact]
public void ImmutableList_ConcurrentReads_ThreadSafe()
{
    // Arrange
    var list = ImmutableList.CreateRange(Enumerable.Range(0, 1000));
    var results = new ConcurrentBag<int>();
    
    // Act
    Parallel.For(0, 100, _ =>
    {
        results.Add(list.Count);
    });
    
    // Assert
    results.Should().AllBe(1000);
}
```

## Guidelines

1. **Use IReadOnlyList**: Expose collections as readonly interfaces
2. **Prefer immutable**: Use ImmutableList for truly unchangeable collections
3. **Defensive copies**: Return copies when mutation would break invariants
4. **Test cast attempts**: Verify collections can't be cast to mutable types
5. **Document behavior**: Clear whether changes to backing store are visible

## Benefits

1. **Prevent bugs**: Immutability prevents accidental modifications
2. **Thread safety**: Immutable collections are inherently thread-safe
3. **Clear intent**: Readonly types signal immutability expectations
4. **Reliable state**: Invariants can't be violated through collection manipulation

## See Also

- [Testing Record Types](./testing-record-types.md) — immutable record types
- [Testing Value Objects](./testing-value-objects.md) — immutable value semantics
- [Testing Concurrency](./testing-concurrency.md) — thread-safe collections
- [Testing Collection Assertions](./testing-collection-assertions.md) — collection testing
