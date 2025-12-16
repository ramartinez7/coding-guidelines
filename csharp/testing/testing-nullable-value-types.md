# Testing Nullable Value Types

> Verify nullable value type behavior, HasValue/Value properties, and null handling with GetValueOrDefault.

## Problem

Nullable value types (`int?`, `DateTime?`, etc.) require testing for null states, value extraction, and default value handling. Tests must verify both the presence and absence of values.

## Pattern

Use FluentAssertions to test nullable value types, checking HasValue, Value, and null coalescing behavior.

## Example

### ❌ Before - Incomplete Nullable Testing

```csharp
[TestMethod]
public void GetAge_ReturnsValue()
{
    var age = customer.GetAge();
    Assert.IsTrue(age.HasValue);
    // ❌ Missing: null case, value verification
}
```

### ✅ After - Complete Nullable Value Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class CustomerAgeTests
{
    [TestMethod]
    public void GetAge_WhenBirthDateProvided_ReturnsAge()
    {
        // Arrange
        var birthDate = DateTime.Now.AddYears(-30);
        var customer = Customer.Create("John", "Doe", birthDate);

        // Act
        var age = customer.GetAge();

        // Assert
        age.Should().HaveValue();
        age.Value.Should().Be(30);
    }

    [TestMethod]
    public void GetAge_WhenBirthDateNull_ReturnsNull()
    {
        // Arrange
        var customer = Customer.Create("John", "Doe", birthDate: null);

        // Act
        var age = customer.GetAge();

        // Assert
        age.Should().NotHaveValue();
        age.Should().BeNull();
    }

    [TestMethod]
    public void GetAge_WithNullBirthDate_GetValueOrDefaultReturnsZero()
    {
        // Arrange
        var customer = Customer.Create("John", "Doe", birthDate: null);

        // Act
        var age = customer.GetAge();
        var ageOrDefault = age.GetValueOrDefault();

        // Assert
        ageOrDefault.Should().Be(0);
    }
}
```

## Testing HasValue and Value Properties

```csharp
public class OrderDiscount
{
    public decimal? DiscountPercentage { get; }
    public Money Total { get; }

    public Money CalculateDiscountedTotal()
    {
        if (!DiscountPercentage.HasValue)
            return Total;

        var discount = Total.Amount * (DiscountPercentage.Value / 100m);
        return Money.Create(Total.Amount - discount, Total.Currency).Value;
    }
}

[TestClass]
public class NullableValuePropertyTests
{
    [TestMethod]
    public void DiscountPercentage_WhenSet_HasValue()
    {
        // Arrange
        var discount = new OrderDiscount(
            total: Money.USD(100m),
            discountPercentage: 10m
        );

        // Act & Assert
        discount.DiscountPercentage.Should().HaveValue();
        discount.DiscountPercentage.Value.Should().Be(10m);
    }

    [TestMethod]
    public void DiscountPercentage_WhenNotSet_HasNoValue()
    {
        // Arrange
        var discount = new OrderDiscount(
            total: Money.USD(100m),
            discountPercentage: null
        );

        // Act & Assert
        discount.DiscountPercentage.Should().NotHaveValue();
        discount.DiscountPercentage.Should().BeNull();
    }

    [TestMethod]
    public void CalculateDiscountedTotal_WithNullDiscount_ReturnsOriginalTotal()
    {
        // Arrange
        var discount = new OrderDiscount(
            total: Money.USD(100m),
            discountPercentage: null
        );

        // Act
        var result = discount.CalculateDiscountedTotal();

        // Assert
        result.Amount.Should().Be(100m);
    }

    [TestMethod]
    public void CalculateDiscountedTotal_WithDiscount_ReturnsReducedTotal()
    {
        // Arrange
        var discount = new OrderDiscount(
            total: Money.USD(100m),
            discountPercentage: 10m
        );

        // Act
        var result = discount.CalculateDiscountedTotal();

        // Assert
        result.Amount.Should().Be(90m);
    }
}
```

## Testing Null Coalescing

```csharp
[TestClass]
public class NullCoalescingTests
{
    [TestMethod]
    public void NullCoalescing_WithValue_ReturnsValue()
    {
        // Arrange
        int? nullable = 42;

        // Act
        var result = nullable ?? 0;

        // Assert
        result.Should().Be(42);
    }

    [TestMethod]
    public void NullCoalescing_WithNull_ReturnsDefault()
    {
        // Arrange
        int? nullable = null;

        // Act
        var result = nullable ?? -1;

        // Assert
        result.Should().Be(-1);
    }

    [TestMethod]
    public void NullCoalescing_Chain_SelectsFirstNonNull()
    {
        // Arrange
        int? first = null;
        int? second = null;
        int? third = 42;

        // Act
        var result = first ?? second ?? third ?? 0;

        // Assert
        result.Should().Be(42);
    }
}
```

## Testing GetValueOrDefault

```csharp
[TestClass]
public class GetValueOrDefaultTests
{
    [TestMethod]
    public void GetValueOrDefault_WithValue_ReturnsValue()
    {
        // Arrange
        int? nullable = 42;

        // Act
        var result = nullable.GetValueOrDefault();

        // Assert
        result.Should().Be(42);
    }

    [TestMethod]
    public void GetValueOrDefault_WithNull_ReturnsTypeDefault()
    {
        // Arrange
        int? nullable = null;

        // Act
        var result = nullable.GetValueOrDefault();

        // Assert
        result.Should().Be(0);
    }

    [TestMethod]
    public void GetValueOrDefault_WithCustomDefault_ReturnsCustom()
    {
        // Arrange
        int? nullable = null;

        // Act
        var result = nullable.GetValueOrDefault(-1);

        // Assert
        result.Should().Be(-1);
    }

    [TestMethod]
    public void DateTime_GetValueOrDefault_ReturnsMinValue()
    {
        // Arrange
        DateTime? nullable = null;

        // Act
        var result = nullable.GetValueOrDefault();

        // Assert
        result.Should().Be(DateTime.MinValue);
    }
}
```

## Testing Comparison Operations

```csharp
[TestClass]
public class NullableComparisonTests
{
    [TestMethod]
    public void NullableEquals_BothNull_ReturnsTrue()
    {
        // Arrange
        int? value1 = null;
        int? value2 = null;

        // Act
        var result = value1 == value2;

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void NullableEquals_OneNull_ReturnsFalse()
    {
        // Arrange
        int? value1 = 42;
        int? value2 = null;

        // Act
        var result = value1 == value2;

        // Assert
        result.Should().BeFalse();
    }

    [TestMethod]
    public void NullableEquals_SameValues_ReturnsTrue()
    {
        // Arrange
        int? value1 = 42;
        int? value2 = 42;

        // Act
        var result = value1 == value2;

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void NullableComparison_WithNull_ReturnsFalse()
    {
        // Arrange
        int? value1 = 42;
        int? value2 = null;

        // Act
        var greaterThan = value1 > value2;
        var lessThan = value1 < value2;

        // Assert
        greaterThan.Should().BeFalse();
        lessThan.Should().BeFalse();
    }
}
```

## Domain-Specific Nullable Testing

```csharp
public record Appointment
{
    public AppointmentId Id { get; }
    public DateTime ScheduledAt { get; }
    public DateTime? CompletedAt { get; }

    public bool IsCompleted => CompletedAt.HasValue;
    public TimeSpan? Duration => 
        CompletedAt.HasValue 
            ? CompletedAt.Value - ScheduledAt 
            : null;
}

[TestClass]
public class AppointmentNullableTests
{
    [TestMethod]
    public void IsCompleted_WhenCompletedAtHasValue_ReturnsTrue()
    {
        // Arrange
        var appointment = CreateAppointment(
            scheduledAt: DateTime.Now,
            completedAt: DateTime.Now.AddHours(1)
        );

        // Act & Assert
        appointment.IsCompleted.Should().BeTrue();
        appointment.CompletedAt.Should().HaveValue();
    }

    [TestMethod]
    public void IsCompleted_WhenCompletedAtNull_ReturnsFalse()
    {
        // Arrange
        var appointment = CreateAppointment(
            scheduledAt: DateTime.Now,
            completedAt: null
        );

        // Act & Assert
        appointment.IsCompleted.Should().BeFalse();
        appointment.CompletedAt.Should().NotHaveValue();
    }

    [TestMethod]
    public void Duration_WhenCompleted_ReturnsTimeSpan()
    {
        // Arrange
        var scheduled = DateTime.Now;
        var completed = scheduled.AddHours(2);
        var appointment = CreateAppointment(scheduled, completed);

        // Act
        var duration = appointment.Duration;

        // Assert
        duration.Should().HaveValue();
        duration.Value.Should().Be(TimeSpan.FromHours(2));
    }

    [TestMethod]
    public void Duration_WhenNotCompleted_ReturnsNull()
    {
        // Arrange
        var appointment = CreateAppointment(
            scheduledAt: DateTime.Now,
            completedAt: null
        );

        // Act
        var duration = appointment.Duration;

        // Assert
        duration.Should().NotHaveValue();
        duration.Should().BeNull();
    }
}
```

## Testing Nullable Conversions

```csharp
[TestClass]
public class NullableConversionTests
{
    [TestMethod]
    public void ImplicitConversion_FromValueToNullable()
    {
        // Arrange
        int value = 42;

        // Act
        int? nullable = value;

        // Assert
        nullable.Should().HaveValue();
        nullable.Value.Should().Be(42);
    }

    [TestMethod]
    public void ExplicitCast_FromNullableToValue_WhenHasValue()
    {
        // Arrange
        int? nullable = 42;

        // Act
        int value = (int)nullable;

        // Assert
        value.Should().Be(42);
    }

    [TestMethod]
    public void ExplicitCast_FromNullableToValue_WhenNull_Throws()
    {
        // Arrange
        int? nullable = null;

        // Act
        Action act = () => { int value = (int)nullable; };

        // Assert
        act.Should().Throw<InvalidOperationException>();
    }
}
```

## Why It's Important

1. **Null Safety**: Explicit handling of missing values
2. **Type Safety**: Compiler-enforced null checking
3. **Clarity**: Clear distinction between "no value" and "zero/default"
4. **Domain Modeling**: Optional fields modeled explicitly
5. **Error Prevention**: Avoid InvalidOperationException

## Guidelines

**Testing Nullable Values**
- `.Should().HaveValue()` - has a value
- `.Should().NotHaveValue()` - is null
- `.Should().BeNull()` - is null (alternative)
- Test both Value and GetValueOrDefault()

**Common Patterns**
- Test null and non-null cases separately
- Verify HasValue before accessing Value
- Test null coalescing behavior
- Verify comparison operations with null

**Avoid**
- Accessing Value without checking HasValue
- Using nullable for required fields
- Comparing nullable to value type directly

## Common Pitfalls

❌ **Accessing Value without check**
```csharp
// Dangerous
var age = customer.BirthDate.Value; // InvalidOperationException if null
```

✅ **Check HasValue first**
```csharp
// Safe
age.Should().HaveValue();
var actualAge = age.Value;
```

❌ **Using GetValueOrDefault without understanding default**
```csharp
// Ambiguous - is 0 intentional or default?
var count = nullable.GetValueOrDefault();
```

✅ **Be explicit about defaults**
```csharp
// Clear
var count = nullable ?? -1; // -1 indicates "no value"
```

## See Also

- [Testing Reference Types and Null](./testing-reference-types-null.md) — reference type null handling
- [Nullable Reference Types](../patterns/nullable-reference-types.md) — C# 8+ nullability
- [Option Monad](../patterns/option-monad.md) — functional optional values
- [Nullability and Optionality](../patterns/nullability-optionality.md) — modeling optional data
