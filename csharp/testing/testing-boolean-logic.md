# Testing Boolean Logic

> Verify boolean values, conditions, and logical operations using clear and expressive assertions.

## Problem

Testing boolean logic often involves multiple conditions, compound expressions, and logical operators. Tests need to clearly express expected truth values and logical relationships.

## Pattern

Use FluentAssertions' boolean assertions to test true/false values, logical implications, and conditional logic expressively.

## Example

### ❌ Before - Unclear Boolean Testing

```csharp
[TestMethod]
public void IsEligible_ChecksConditions()
{
    var result = service.IsEligible(customer);
    Assert.IsTrue(result == true);
    Assert.IsFalse(result == false);
    // ❌ Redundant and unclear
}
```

### ✅ After - Clear Boolean Assertions

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class CustomerEligibilityTests
{
    [TestMethod]
    public void IsEligible_WithValidCustomer_ReturnsTrue()
    {
        // Arrange
        var customer = Customer.Create("John", "Doe", age: 25);
        var service = new EligibilityService();

        // Act
        var result = service.IsEligible(customer);

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void IsEligible_WithInvalidCustomer_ReturnsFalse()
    {
        // Arrange
        var customer = Customer.Create("John", "Doe", age: 15);
        var service = new EligibilityService();

        // Act
        var result = service.IsEligible(customer);

        // Assert
        result.Should().BeFalse();
    }

    [TestMethod]
    public void IsEligible_WithNullCustomer_ReturnsFalse()
    {
        // Arrange
        var service = new EligibilityService();

        // Act
        var result = service.IsEligible(null);

        // Assert
        result.Should().BeFalse();
    }
}
```

## Testing Boolean Properties

```csharp
public record Order
{
    public OrderId Id { get; }
    public OrderStatus Status { get; }
    public Money Total { get; }

    public bool IsCompleted => Status == OrderStatus.Completed;
    public bool IsPaid => Total.Amount > 0 && Status != OrderStatus.Cancelled;
    public bool CanBeCancelled => !IsCompleted && !IsPaid;

    // Constructor omitted for brevity
}

[TestClass]
public class OrderBooleanPropertyTests
{
    [TestMethod]
    public void IsCompleted_WhenStatusCompleted_ReturnsTrue()
    {
        // Arrange
        var order = CreateOrder(status: OrderStatus.Completed);

        // Act & Assert
        order.IsCompleted.Should().BeTrue();
    }

    [TestMethod]
    public void IsCompleted_WhenStatusPending_ReturnsFalse()
    {
        // Arrange
        var order = CreateOrder(status: OrderStatus.Pending);

        // Act & Assert
        order.IsCompleted.Should().BeFalse();
    }

    [TestMethod]
    public void CanBeCancelled_WhenNotCompletedAndNotPaid_ReturnsTrue()
    {
        // Arrange
        var order = CreateOrder(
            status: OrderStatus.Pending,
            total: Money.USD(0m)
        );

        // Act & Assert
        order.CanBeCancelled.Should().BeTrue();
        order.IsCompleted.Should().BeFalse();
        order.IsPaid.Should().BeFalse();
    }
}
```

## Testing Logical Operators

```csharp
[TestClass]
public class LogicalOperatorTests
{
    [TestMethod]
    public void And_BothTrue_ReturnsTrue()
    {
        // Arrange
        var condition1 = true;
        var condition2 = true;

        // Act
        var result = condition1 && condition2;

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void And_OneFalse_ReturnsFalse()
    {
        // Arrange
        var condition1 = true;
        var condition2 = false;

        // Act
        var result = condition1 && condition2;

        // Assert
        result.Should().BeFalse();
    }

    [TestMethod]
    public void Or_OneTrue_ReturnsTrue()
    {
        // Arrange
        var condition1 = true;
        var condition2 = false;

        // Act
        var result = condition1 || condition2;

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void Not_InvertsBoolean()
    {
        // Arrange
        var value = true;

        // Act
        var result = !value;

        // Assert
        result.Should().BeFalse();
        (!result).Should().BeTrue();
    }
}
```

## Testing Guard Clauses

```csharp
public class OrderProcessor
{
    public Result Process(Order order)
    {
        if (order == null)
            return Result.Failure("Order cannot be null");

        if (order.IsCompleted)
            return Result.Failure("Order is already completed");

        if (!order.CanBeCancelled)
            return Result.Failure("Order cannot be processed");

        // Processing logic...
        return Result.Success();
    }
}

[TestClass]
public class OrderProcessorGuardTests
{
    [TestMethod]
    public void Process_WithNullOrder_ReturnsFailure()
    {
        // Arrange
        var processor = new OrderProcessor();

        // Act
        var result = processor.Process(null);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.IsSuccess.Should().BeFalse();
        result.Error.Should().Contain("null");
    }

    [TestMethod]
    public void Process_WithCompletedOrder_ReturnsFailure()
    {
        // Arrange
        var order = CreateOrder(status: OrderStatus.Completed);
        var processor = new OrderProcessor();

        // Act
        var result = processor.Process(order);

        // Assert
        result.IsFailure.Should().BeTrue();
        order.IsCompleted.Should().BeTrue();
    }

    [TestMethod]
    public void Process_WithValidOrder_ReturnsSuccess()
    {
        // Arrange
        var order = CreateOrder(status: OrderStatus.Pending);
        var processor = new OrderProcessor();

        // Act
        var result = processor.Process(order);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.IsFailure.Should().BeFalse();
    }
}
```

## Testing Conditional Logic

```csharp
public class DiscountCalculator
{
    public bool IsEligibleForDiscount(Customer customer, decimal orderTotal)
    {
        var isVip = customer.Status == CustomerStatus.VIP;
        var hasLargeOrder = orderTotal >= 1000m;
        var isLongTermCustomer = customer.MemberSince.AddYears(2) <= DateTime.Now;

        return isVip || (hasLargeOrder && isLongTermCustomer);
    }
}

[TestClass]
public class DiscountEligibilityTests
{
    [TestMethod]
    public void IsEligibleForDiscount_VipCustomer_ReturnsTrue()
    {
        // Arrange
        var customer = CreateCustomer(status: CustomerStatus.VIP);
        var calculator = new DiscountCalculator();

        // Act
        var result = calculator.IsEligibleForDiscount(customer, orderTotal: 50m);

        // Assert
        result.Should().BeTrue("VIP customers always get discounts");
    }

    [TestMethod]
    public void IsEligibleForDiscount_LargeOrderAndLongTerm_ReturnsTrue()
    {
        // Arrange
        var customer = CreateCustomer(
            status: CustomerStatus.Regular,
            memberSince: DateTime.Now.AddYears(-3)
        );
        var calculator = new DiscountCalculator();

        // Act
        var result = calculator.IsEligibleForDiscount(customer, orderTotal: 1500m);

        // Assert
        result.Should().BeTrue("large order + long-term membership qualifies");
    }

    [TestMethod]
    public void IsEligibleForDiscount_SmallOrderNewCustomer_ReturnsFalse()
    {
        // Arrange
        var customer = CreateCustomer(
            status: CustomerStatus.Regular,
            memberSince: DateTime.Now.AddMonths(-3)
        );
        var calculator = new DiscountCalculator();

        // Act
        var result = calculator.IsEligibleForDiscount(customer, orderTotal: 100m);

        // Assert
        result.Should().BeFalse("doesn't meet any discount criteria");
    }
}
```

## Testing Boolean Flags and States

```csharp
public class FeatureFlags
{
    public bool IsEnabled(string featureName) => 
        _enabledFeatures.Contains(featureName);

    public bool AreAllEnabled(params string[] features) =>
        features.All(IsEnabled);

    public bool IsAnyEnabled(params string[] features) =>
        features.Any(IsEnabled);
}

[TestClass]
public class FeatureFlagTests
{
    [TestMethod]
    public void IsEnabled_ForEnabledFeature_ReturnsTrue()
    {
        // Arrange
        var flags = new FeatureFlags();
        flags.Enable("NewCheckout");

        // Act
        var result = flags.IsEnabled("NewCheckout");

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void AreAllEnabled_WhenAllEnabled_ReturnsTrue()
    {
        // Arrange
        var flags = new FeatureFlags();
        flags.Enable("Feature1");
        flags.Enable("Feature2");

        // Act
        var result = flags.AreAllEnabled("Feature1", "Feature2");

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void AreAllEnabled_WhenOneDisabled_ReturnsFalse()
    {
        // Arrange
        var flags = new FeatureFlags();
        flags.Enable("Feature1");

        // Act
        var result = flags.AreAllEnabled("Feature1", "Feature2");

        // Assert
        result.Should().BeFalse();
    }

    [TestMethod]
    public void IsAnyEnabled_WhenOneEnabled_ReturnsTrue()
    {
        // Arrange
        var flags = new FeatureFlags();
        flags.Enable("Feature1");

        // Act
        var result = flags.IsAnyEnabled("Feature1", "Feature2");

        // Assert
        result.Should().BeTrue();
    }
}
```

## Why It's Important

1. **Clarity**: Express boolean intent clearly
2. **Readability**: Tests read like specifications
3. **Debugging**: Clear assertions help identify logic errors
4. **Completeness**: Test both true and false cases
5. **Guard Clauses**: Verify preconditions are enforced

## Guidelines

**Boolean Assertions**
- `.Should().BeTrue()` - assert true
- `.Should().BeFalse()` - assert false
- Add reason strings for complex conditions
- Test both positive and negative cases

**Testing Patterns**
- Test each boolean condition independently
- Test combined logical expressions
- Verify guard clauses prevent invalid states
- Check edge cases and boundary conditions

**Avoid**
- Comparing boolean to true/false: `result == true`
- Double negatives: `(!result).Should().BeFalse()`
- Testing implementation instead of behavior

## Common Pitfalls

❌ **Redundant comparisons**
```csharp
// Redundant
(result == true).Should().BeTrue();
Assert.IsTrue(result == true);
```

✅ **Direct boolean assertion**
```csharp
// Clear
result.Should().BeTrue();
```

❌ **Testing only true case**
```csharp
// Incomplete
isEligible.Should().BeTrue();
```

✅ **Test both cases**
```csharp
// Complete
eligibleCustomer.IsEligible().Should().BeTrue();
ineligibleCustomer.IsEligible().Should().BeFalse();
```

## See Also

- [Testing Guard Clauses](./testing-guard-clauses.md) — guard clause patterns
- [Boolean Blindness](../patterns/boolean-blindness.md) — avoiding boolean problems
- [Flag Arguments](../patterns/flag-arguments.md) — boolean parameter issues
- [Honest Functions](../patterns/honest-functions.md) — explicit return types
