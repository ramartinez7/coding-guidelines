# Testing Collection Assertions

> Verify collection contents, order, and properties using FluentAssertions' powerful collection testing capabilities.

## Problem

Collections require specific assertions beyond simple equality checks. Tests need to verify element presence, ordering, uniqueness, and collection properties without writing verbose assertion logic.

## Pattern

Use FluentAssertions' collection assertions to expressively test collection contents, ordering, filtering, and properties.

## Example

### ❌ Before - Verbose Collection Testing

```csharp
[TestMethod]
public void GetOrders_ReturnsExpectedOrders()
{
    var orders = service.GetOrders();
    Assert.IsTrue(orders.Count() == 3);
    Assert.IsTrue(orders.Any(o => o.Id == orderId1));
    Assert.IsTrue(orders.Any(o => o.Id == orderId2));
    // ❌ Verbose and hard to read
}
```

### ✅ After - Fluent Collection Assertions

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class OrderServiceCollectionTests
{
    [TestMethod]
    public void GetOrders_ReturnsExpectedOrders()
    {
        // Arrange
        var orderId1 = OrderId.New();
        var orderId2 = OrderId.New();
        var orderId3 = OrderId.New();
        var service = CreateServiceWithOrders(orderId1, orderId2, orderId3);

        // Act
        var orders = service.GetOrders();

        // Assert
        orders.Should().HaveCount(3);
        orders.Should().Contain(o => o.Id == orderId1);
        orders.Should().Contain(o => o.Id == orderId2);
        orders.Should().Contain(o => o.Id == orderId3);
    }

    [TestMethod]
    public void GetOrders_ReturnsOrdersInCreationOrder()
    {
        // Arrange
        var orderId1 = OrderId.New();
        var orderId2 = OrderId.New();
        var service = CreateServiceWithOrders(orderId1, orderId2);

        // Act
        var orders = service.GetOrders().ToList();

        // Assert
        orders.Should().HaveCount(2);
        orders[0].Id.Should().Be(orderId1);
        orders[1].Id.Should().Be(orderId2);
    }

    [TestMethod]
    public void GetOrders_WhenNoOrders_ReturnsEmptyCollection()
    {
        // Arrange
        var service = CreateServiceWithOrders();

        // Act
        var orders = service.GetOrders();

        // Assert
        orders.Should().BeEmpty();
        orders.Should().NotBeNull();
    }
}
```

## Collection Content Assertions

```csharp
[TestClass]
public class CollectionContentTests
{
    [TestMethod]
    public void Collection_ContainsExpectedElements()
    {
        // Arrange
        var numbers = new[] { 1, 2, 3, 4, 5 };

        // Act & Assert
        numbers.Should().Contain(3);
        numbers.Should().Contain(new[] { 2, 4 });
        numbers.Should().NotContain(10);
        numbers.Should().NotContain(new[] { 7, 8, 9 });
    }

    [TestMethod]
    public void Collection_ContainsItemsMatchingPredicate()
    {
        // Arrange
        var orders = new[]
        {
            Order.Create(OrderId.New(), CustomerId.New(), Money.USD(100m)),
            Order.Create(OrderId.New(), CustomerId.New(), Money.USD(200m)),
            Order.Create(OrderId.New(), CustomerId.New(), Money.USD(50m))
        };

        // Act & Assert
        orders.Should().Contain(o => o.Total.Amount > 150m);
        orders.Should().NotContain(o => o.Total.Amount > 300m);
    }

    [TestMethod]
    public void Collection_OnlyContainsExpectedItems()
    {
        // Arrange
        var validStatuses = new[] { "Pending", "Processing", "Completed" };

        // Act & Assert
        validStatuses.Should().OnlyContain(s => s.Length > 5);
        validStatuses.Should().NotContainNulls();
    }

    [TestMethod]
    public void Collection_AllItemsSatisfyCondition()
    {
        // Arrange
        var positiveNumbers = new[] { 1, 2, 3, 4, 5 };

        // Act & Assert
        positiveNumbers.Should().OnlyContain(n => n > 0);
        positiveNumbers.Should().AllSatisfy(n => n.Should().BePositive());
    }
}
```

## Collection Ordering and Uniqueness

```csharp
[TestClass]
public class CollectionOrderingTests
{
    [TestMethod]
    public void Collection_IsInAscendingOrder()
    {
        // Arrange
        var numbers = new[] { 1, 2, 3, 4, 5 };

        // Act & Assert
        numbers.Should().BeInAscendingOrder();
    }

    [TestMethod]
    public void Collection_IsInDescendingOrder()
    {
        // Arrange
        var numbers = new[] { 5, 4, 3, 2, 1 };

        // Act & Assert
        numbers.Should().BeInDescendingOrder();
    }

    [TestMethod]
    public void Collection_OrderedByProperty()
    {
        // Arrange
        var orders = new[]
        {
            Order.Create(OrderId.New(), CustomerId.New(), Money.USD(50m)),
            Order.Create(OrderId.New(), CustomerId.New(), Money.USD(100m)),
            Order.Create(OrderId.New(), CustomerId.New(), Money.USD(200m))
        };

        // Act & Assert
        orders.Should().BeInAscendingOrder(o => o.Total.Amount);
    }

    [TestMethod]
    public void Collection_ContainsNoDuplicates()
    {
        // Arrange
        var numbers = new[] { 1, 2, 3, 4, 5 };

        // Act & Assert
        numbers.Should().OnlyHaveUniqueItems();
    }
}
```

## Exact Collection Matching

```csharp
[TestClass]
public class ExactCollectionMatchingTests
{
    [TestMethod]
    public void Collection_ExactlyMatchesExpected()
    {
        // Arrange
        var actual = new[] { 1, 2, 3 };
        var expected = new[] { 1, 2, 3 };

        // Act & Assert
        actual.Should().Equal(expected);
    }

    [TestMethod]
    public void Collection_MatchesInAnyOrder()
    {
        // Arrange
        var actual = new[] { 3, 1, 2 };
        var expected = new[] { 1, 2, 3 };

        // Act & Assert
        actual.Should().BeEquivalentTo(expected);
    }

    [TestMethod]
    public void Collection_MatchesSubset()
    {
        // Arrange
        var actual = new[] { 1, 2, 3, 4, 5 };
        var subset = new[] { 2, 4 };

        // Act & Assert
        actual.Should().Contain(subset);
    }

    [TestMethod]
    public void Collection_StartsWithExpectedSequence()
    {
        // Arrange
        var numbers = new[] { 1, 2, 3, 4, 5 };

        // Act & Assert
        numbers.Should().StartWith(new[] { 1, 2 });
        numbers.Should().EndWith(new[] { 4, 5 });
    }
}
```

## Collection Size and Null Handling

```csharp
[TestClass]
public class CollectionSizeTests
{
    [TestMethod]
    public void Collection_HasExpectedSize()
    {
        // Arrange
        var items = new[] { 1, 2, 3 };

        // Act & Assert
        items.Should().HaveCount(3);
        items.Should().NotBeEmpty();
        items.Should().HaveCountGreaterThan(2);
        items.Should().HaveCountLessThanOrEqualTo(5);
    }

    [TestMethod]
    public void Collection_DoesNotContainNulls()
    {
        // Arrange
        var items = new[] { "a", "b", "c" };

        // Act & Assert
        items.Should().NotContainNulls();
    }

    [TestMethod]
    public void Collection_ContainsOnlySingleElement()
    {
        // Arrange
        var items = new[] { 42 };

        // Act & Assert
        items.Should().ContainSingle();
        items.Should().ContainSingle(x => x == 42);
    }
}
```

## Complex Object Collection Testing

```csharp
[TestClass]
public class ComplexCollectionTests
{
    [TestMethod]
    public void Collection_MatchesExpectedObjects()
    {
        // Arrange
        var orders = new[]
        {
            new { Id = 1, Total = 100m },
            new { Id = 2, Total = 200m }
        };

        // Act & Assert
        orders.Should().BeEquivalentTo(new[]
        {
            new { Id = 1, Total = 100m },
            new { Id = 2, Total = 200m }
        });
    }

    [TestMethod]
    public void Collection_MatchesSelectedProperties()
    {
        // Arrange
        var orders = new[]
        {
            Order.Create(OrderId.New(), CustomerId.New(), Money.USD(100m)),
            Order.Create(OrderId.New(), CustomerId.New(), Money.USD(200m))
        };

        // Act & Assert
        orders.Select(o => o.Total.Amount)
              .Should().Equal(new[] { 100m, 200m });
    }

    [TestMethod]
    public void Collection_SatisfiesComplexConditions()
    {
        // Arrange
        var orders = new[]
        {
            Order.Create(OrderId.New(), CustomerId.New(), Money.USD(100m)),
            Order.Create(OrderId.New(), CustomerId.New(), Money.USD(200m))
        };

        // Act & Assert
        orders.Should().SatisfyRespectively(
            first => first.Total.Amount.Should().Be(100m),
            second => second.Total.Amount.Should().Be(200m)
        );
    }
}
```

## Why It's Important

1. **Expressiveness**: FluentAssertions makes test intent clear
2. **Better Error Messages**: Failures show exactly what was expected vs actual
3. **Less Code**: Reduces boilerplate assertion logic
4. **Maintainability**: Easier to read and modify tests
5. **Type Safety**: Compile-time checking of assertions

## Guidelines

**Common Collection Assertions**
- `.Should().HaveCount(n)` - exact count
- `.Should().Contain(item)` - contains specific item
- `.Should().BeEmpty()` / `.NotBeEmpty()` - empty check
- `.Should().BeEquivalentTo(expected)` - unordered match
- `.Should().Equal(expected)` - ordered match
- `.Should().OnlyHaveUniqueItems()` - no duplicates
- `.Should().BeInAscendingOrder()` - ordering

**When to Use Each**
- Use `.Equal()` when order matters
- Use `.BeEquivalentTo()` when order doesn't matter
- Use `.Contain()` for partial matches
- Use `.OnlyContain()` for all elements matching condition

## Common Pitfalls

❌ **Using wrong equality method**
```csharp
// Wrong - tests reference equality
actual.Should().BeSameAs(expected);
```

✅ **Use value equality**
```csharp
// Correct - tests value equality
actual.Should().BeEquivalentTo(expected);
```

❌ **Not handling null collections**
```csharp
// Missing null check
result.Should().HaveCount(0);
```

✅ **Explicit null and empty checks**
```csharp
// Clear intent
result.Should().NotBeNull();
result.Should().BeEmpty();
```

## See Also

- [Testing LINQ Queries](./testing-linq-queries.md) — query result testing
- [Parameterized Tests](./parameterized-tests.md) — testing multiple collections
- [Testing Domain Invariants](./testing-domain-invariants.md) — collection invariants
- [First-Class Collections](../patterns/first-class-collections.md) — domain collections
