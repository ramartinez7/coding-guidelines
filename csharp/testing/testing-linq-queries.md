# Testing LINQ Queries

> Verify LINQ query results, transformations, filtering, and projections using FluentAssertions.

## Problem

LINQ queries involve complex transformations, filtering, and projections. Tests need to verify query logic, result ordering, and transformations without executing the query multiple times.

## Pattern

Use FluentAssertions to test LINQ query results, materialize queries once, and verify transformations and filtering logic.

## Example

### ❌ Before - Multiple Query Execution

```csharp
[TestMethod]
public void GetActiveCustomers_FiltersCorrectly()
{
    var customers = service.GetActiveCustomers();
    Assert.IsTrue(customers.Count() > 0);
    Assert.IsTrue(customers.All(c => c.IsActive));
    Assert.IsTrue(customers.Any(c => c.Id == customerId));
    // ❌ Query executed 3 times!
}
```

### ✅ After - Single Materialization

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class CustomerQueryTests
{
    [TestMethod]
    public void GetActiveCustomers_ReturnsOnlyActiveCustomers()
    {
        // Arrange
        var service = CreateServiceWithMixedCustomers();

        // Act
        var customers = service.GetActiveCustomers().ToList();

        // Assert
        customers.Should().NotBeEmpty();
        customers.Should().OnlyContain(c => c.IsActive);
    }

    [TestMethod]
    public void GetActiveCustomers_OrdersByName()
    {
        // Arrange
        var service = CreateServiceWithCustomers();

        // Act
        var customers = service.GetActiveCustomers().ToList();

        // Assert
        customers.Should().BeInAscendingOrder(c => c.LastName)
                          .And.BeInAscendingOrder(c => c.FirstName);
    }
}
```

## Testing Filtering Operations

```csharp
public class OrderQueryService
{
    public IEnumerable<Order> GetOrdersByStatus(OrderStatus status) =>
        _repository.GetAll().Where(o => o.Status == status);

    public IEnumerable<Order> GetOrdersAboveAmount(decimal minAmount) =>
        _repository.GetAll().Where(o => o.Total.Amount > minAmount);

    public IEnumerable<Order> GetRecentOrders(int days) =>
        _repository.GetAll()
            .Where(o => o.CreatedAt >= DateTime.Now.AddDays(-days));
}

[TestClass]
public class OrderFilteringTests
{
    [TestMethod]
    public void GetOrdersByStatus_ReturnsOnlyMatchingStatus()
    {
        // Arrange
        var service = CreateServiceWithOrders();

        // Act
        var orders = service.GetOrdersByStatus(OrderStatus.Pending).ToList();

        // Assert
        orders.Should().NotBeEmpty();
        orders.Should().OnlyContain(o => o.Status == OrderStatus.Pending);
    }

    [TestMethod]
    public void GetOrdersAboveAmount_ReturnsOnlyHighValueOrders()
    {
        // Arrange
        var service = CreateServiceWithOrders();
        var minAmount = 1000m;

        // Act
        var orders = service.GetOrdersAboveAmount(minAmount).ToList();

        // Assert
        orders.Should().NotBeEmpty();
        orders.Should().OnlyContain(o => o.Total.Amount > minAmount);
        orders.Should().AllSatisfy(o => 
            o.Total.Amount.Should().BeGreaterThan(minAmount)
        );
    }

    [TestMethod]
    public void GetRecentOrders_ReturnsOrdersWithinDateRange()
    {
        // Arrange
        var service = CreateServiceWithOrders();
        var days = 7;
        var cutoffDate = DateTime.Now.AddDays(-days);

        // Act
        var orders = service.GetRecentOrders(days).ToList();

        // Assert
        orders.Should().NotBeEmpty();
        orders.Should().OnlyContain(o => o.CreatedAt >= cutoffDate);
    }
}
```

## Testing Projection and Transformation

```csharp
public class CustomerProjectionService
{
    public IEnumerable<CustomerSummary> GetCustomerSummaries() =>
        _repository.GetAll()
            .Select(c => new CustomerSummary
            {
                FullName = $"{c.FirstName} {c.LastName}",
                Email = c.Email,
                OrderCount = c.Orders.Count,
                TotalSpent = c.Orders.Sum(o => o.Total.Amount)
            });
}

[TestClass]
public class ProjectionTests
{
    [TestMethod]
    public void GetCustomerSummaries_ProjectsCorrectly()
    {
        // Arrange
        var service = CreateServiceWithCustomers();

        // Act
        var summaries = service.GetCustomerSummaries().ToList();

        // Assert
        summaries.Should().NotBeEmpty();
        summaries.Should().AllSatisfy(s =>
        {
            s.FullName.Should().NotBeNullOrEmpty();
            s.Email.Should().NotBeNullOrEmpty();
            s.OrderCount.Should().BeGreaterThanOrEqualTo(0);
            s.TotalSpent.Should().BeGreaterThanOrEqualTo(0);
        });
    }

    [TestMethod]
    public void GetCustomerSummaries_CalculatesTotalSpentCorrectly()
    {
        // Arrange
        var customerId = CustomerId.New();
        var service = CreateServiceWithCustomerAndOrders(
            customerId,
            orderAmounts: new[] { 100m, 200m, 300m }
        );

        // Act
        var summary = service.GetCustomerSummaries()
            .First(s => s.CustomerId == customerId);

        // Assert
        summary.TotalSpent.Should().Be(600m);
        summary.OrderCount.Should().Be(3);
    }
}
```

## Testing Aggregation Operations

```csharp
[TestClass]
public class AggregationTests
{
    [TestMethod]
    public void Sum_CalculatesTotalCorrectly()
    {
        // Arrange
        var amounts = new[] { 100m, 200m, 300m };

        // Act
        var total = amounts.Sum();

        // Assert
        total.Should().Be(600m);
    }

    [TestMethod]
    public void GroupBy_GroupsCorrectly()
    {
        // Arrange
        var orders = CreateOrders();

        // Act
        var grouped = orders.GroupBy(o => o.Status).ToList();

        // Assert
        grouped.Should().NotBeEmpty();
        grouped.Should().OnlyHaveUniqueItems(g => g.Key);
        grouped.Should().AllSatisfy(g =>
        {
            g.Should().NotBeEmpty();
            g.Should().OnlyContain(o => o.Status == g.Key);
        });
    }

    [TestMethod]
    public void Average_CalculatesCorrectly()
    {
        // Arrange
        var values = new[] { 10, 20, 30, 40, 50 };

        // Act
        var average = values.Average();

        // Assert
        average.Should().Be(30);
    }

    [TestMethod]
    public void Max_ReturnsLargestValue()
    {
        // Arrange
        var orders = CreateOrders();

        // Act
        var maxAmount = orders.Max(o => o.Total.Amount);

        // Assert
        maxAmount.Should().BeGreaterThan(0);
        orders.Should().Contain(o => o.Total.Amount == maxAmount);
    }
}
```

## Testing Ordering Operations

```csharp
[TestClass]
public class OrderingTests
{
    [TestMethod]
    public void OrderBy_SortsAscending()
    {
        // Arrange
        var numbers = new[] { 3, 1, 4, 1, 5, 9, 2, 6 };

        // Act
        var sorted = numbers.OrderBy(n => n).ToList();

        // Assert
        sorted.Should().BeInAscendingOrder();
        sorted.First().Should().Be(1);
        sorted.Last().Should().Be(9);
    }

    [TestMethod]
    public void OrderByDescending_SortsDescending()
    {
        // Arrange
        var orders = CreateOrders();

        // Act
        var sorted = orders.OrderByDescending(o => o.Total.Amount).ToList();

        // Assert
        sorted.Should().BeInDescendingOrder(o => o.Total.Amount);
    }

    [TestMethod]
    public void ThenBy_PerformsSecondarySort()
    {
        // Arrange
        var customers = CreateCustomers();

        // Act
        var sorted = customers
            .OrderBy(c => c.LastName)
            .ThenBy(c => c.FirstName)
            .ToList();

        // Assert
        sorted.Should().BeInAscendingOrder(c => c.LastName);
        // Verify secondary sort for customers with same last name
        var smiths = sorted.Where(c => c.LastName == "Smith").ToList();
        smiths.Should().BeInAscendingOrder(c => c.FirstName);
    }
}
```

## Testing Set Operations

```csharp
[TestClass]
public class SetOperationTests
{
    [TestMethod]
    public void Distinct_RemovesDuplicates()
    {
        // Arrange
        var numbers = new[] { 1, 2, 2, 3, 3, 3, 4, 5, 5 };

        // Act
        var distinct = numbers.Distinct().ToList();

        // Assert
        distinct.Should().OnlyHaveUniqueItems();
        distinct.Should().Equal(1, 2, 3, 4, 5);
    }

    [TestMethod]
    public void Union_CombinesAndRemovesDuplicates()
    {
        // Arrange
        var set1 = new[] { 1, 2, 3 };
        var set2 = new[] { 3, 4, 5 };

        // Act
        var union = set1.Union(set2).ToList();

        // Assert
        union.Should().OnlyHaveUniqueItems();
        union.Should().BeEquivalentTo(new[] { 1, 2, 3, 4, 5 });
    }

    [TestMethod]
    public void Intersect_ReturnsCommonElements()
    {
        // Arrange
        var set1 = new[] { 1, 2, 3, 4 };
        var set2 = new[] { 3, 4, 5, 6 };

        // Act
        var intersection = set1.Intersect(set2).ToList();

        // Assert
        intersection.Should().BeEquivalentTo(new[] { 3, 4 });
    }

    [TestMethod]
    public void Except_ReturnsElementsNotInSecondSet()
    {
        // Arrange
        var set1 = new[] { 1, 2, 3, 4 };
        var set2 = new[] { 3, 4 };

        // Act
        var difference = set1.Except(set2).ToList();

        // Assert
        difference.Should().BeEquivalentTo(new[] { 1, 2 });
    }
}
```

## Testing Partitioning Operations

```csharp
[TestClass]
public class PartitioningTests
{
    [TestMethod]
    public void Take_ReturnsFirstNElements()
    {
        // Arrange
        var numbers = new[] { 1, 2, 3, 4, 5 };

        // Act
        var taken = numbers.Take(3).ToList();

        // Assert
        taken.Should().HaveCount(3);
        taken.Should().Equal(1, 2, 3);
    }

    [TestMethod]
    public void Skip_SkipsFirstNElements()
    {
        // Arrange
        var numbers = new[] { 1, 2, 3, 4, 5 };

        // Act
        var skipped = numbers.Skip(2).ToList();

        // Assert
        skipped.Should().HaveCount(3);
        skipped.Should().Equal(3, 4, 5);
    }

    [TestMethod]
    public void TakeWhile_TakesUntilConditionFails()
    {
        // Arrange
        var numbers = new[] { 1, 2, 3, 4, 5, 2, 1 };

        // Act
        var taken = numbers.TakeWhile(n => n < 4).ToList();

        // Assert
        taken.Should().Equal(1, 2, 3);
    }

    [TestMethod]
    public void SkipWhile_SkipsUntilConditionFails()
    {
        // Arrange
        var numbers = new[] { 1, 2, 3, 4, 5 };

        // Act
        var skipped = numbers.SkipWhile(n => n < 3).ToList();

        // Assert
        skipped.Should().Equal(3, 4, 5);
    }
}
```

## Testing Quantifier Operations

```csharp
[TestClass]
public class QuantifierTests
{
    [TestMethod]
    public void Any_WithMatchingElement_ReturnsTrue()
    {
        // Arrange
        var numbers = new[] { 1, 2, 3, 4, 5 };

        // Act
        var hasEven = numbers.Any(n => n % 2 == 0);

        // Assert
        hasEven.Should().BeTrue();
    }

    [TestMethod]
    public void All_WhenAllMatch_ReturnsTrue()
    {
        // Arrange
        var positiveNumbers = new[] { 1, 2, 3, 4, 5 };

        // Act
        var allPositive = positiveNumbers.All(n => n > 0);

        // Assert
        allPositive.Should().BeTrue();
    }

    [TestMethod]
    public void Contains_WithElement_ReturnsTrue()
    {
        // Arrange
        var numbers = new[] { 1, 2, 3, 4, 5 };

        // Act
        var contains = numbers.Contains(3);

        // Assert
        contains.Should().BeTrue();
    }
}
```

## Why It's Important

1. **Performance**: Materialize queries once to avoid multiple enumerations
2. **Correctness**: Verify transformations produce expected results
3. **Clarity**: Test query logic separately from data access
4. **Maintainability**: Clear assertions for complex queries
5. **Deferred Execution**: Understand when queries are executed

## Guidelines

**Best Practices**
- Materialize queries with `.ToList()` or `.ToArray()` before assertions
- Test filtering, projection, and aggregation separately
- Verify ordering explicitly when it matters
- Use `.OnlyContain()` for filter verification
- Test edge cases (empty results, single item, etc.)

**Common LINQ Assertions**
- `.Should().OnlyContain(predicate)` - all items match
- `.Should().BeInAscendingOrder()` - sorted correctly
- `.Should().OnlyHaveUniqueItems()` - no duplicates
- `.Should().AllSatisfy(action)` - verify all items

## Common Pitfalls

❌ **Multiple enumeration**
```csharp
// Query executed 3 times!
customers.Should().NotBeEmpty();
customers.Should().HaveCount(5);
customers.Should().OnlyContain(c => c.IsActive);
```

✅ **Single materialization**
```csharp
// Query executed once
var list = customers.ToList();
list.Should().HaveCount(5);
list.Should().OnlyContain(c => c.IsActive);
```

❌ **Not testing ordering**
```csharp
// Order not verified
var result = query.OrderBy(x => x.Name).ToList();
result.Should().HaveCount(5);
```

✅ **Explicit ordering verification**
```csharp
// Order verified
var result = query.OrderBy(x => x.Name).ToList();
result.Should().BeInAscendingOrder(x => x.Name);
```

## See Also

- [Testing Collection Assertions](./testing-collection-assertions.md) — collection testing
- [Parameterized Tests](./parameterized-tests.md) — testing multiple queries
- [Property-Based Testing](./property-based-testing.md) — LINQ properties
- [First-Class Collections](../patterns/first-class-collections.md) — domain collections
