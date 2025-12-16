# Testing Numeric Assertions

> Verify numeric values, ranges, precision, and arithmetic operations using FluentAssertions' numeric testing capabilities.

## Problem

Numeric testing involves various scenarios: exact values, ranges, precision for floating-point numbers, and sign checks. Manual comparisons are verbose and floating-point comparisons are error-prone.

## Pattern

Use FluentAssertions' numeric assertions to test values, ranges, precision, and mathematical properties expressively and accurately.

## Example

### ❌ Before - Manual Numeric Testing

```csharp
[TestMethod]
public void CalculateTotal_ReturnsCorrectAmount()
{
    var total = calculator.Calculate(100m, 0.15m);
    Assert.IsTrue(total >= 114.99m && total <= 115.01m);
    Assert.IsTrue(total > 0);
    // ❌ Verbose and unclear precision handling
}
```

### ✅ After - Fluent Numeric Assertions

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class PriceCalculatorTests
{
    [TestMethod]
    public void CalculateTotal_WithTax_ReturnsCorrectAmount()
    {
        // Arrange
        var calculator = new PriceCalculator();
        var subtotal = 100m;
        var taxRate = 0.15m;

        // Act
        var total = calculator.CalculateTotal(subtotal, taxRate);

        // Assert
        total.Should().Be(115m);
        total.Should().BePositive();
        total.Should().BeGreaterThan(subtotal);
    }

    [TestMethod]
    public void CalculateDiscount_ReturnsValueInExpectedRange()
    {
        // Arrange
        var calculator = new PriceCalculator();
        var price = 100m;

        // Act
        var discount = calculator.CalculateDiscount(price);

        // Assert
        discount.Should().BeInRange(10m, 20m);
        discount.Should().BeLessThan(price);
    }
}
```

## Exact Value Assertions

```csharp
[TestClass]
public class ExactValueTests
{
    [TestMethod]
    public void Integer_HasExpectedValue()
    {
        // Arrange
        var count = 42;

        // Act & Assert
        count.Should().Be(42);
        count.Should().NotBe(0);
    }

    [TestMethod]
    public void Decimal_HasExactValue()
    {
        // Arrange
        var price = 19.99m;

        // Act & Assert
        price.Should().Be(19.99m);
    }

    [TestMethod]
    public void Double_ApproximatelyEquals_WithPrecision()
    {
        // Arrange
        var calculated = 1.0 / 3.0;
        var expected = 0.333333;

        // Act & Assert
        calculated.Should().BeApproximately(expected, precision: 0.000001);
    }
}
```

## Range and Comparison Assertions

```csharp
[TestClass]
public class RangeTests
{
    [TestMethod]
    public void Value_IsInExpectedRange()
    {
        // Arrange
        var temperature = 25;

        // Act & Assert
        temperature.Should().BeInRange(20, 30);
        temperature.Should().NotBeInRange(0, 10);
    }

    [TestMethod]
    public void Value_IsGreaterOrLessThan()
    {
        // Arrange
        var age = 25;

        // Act & Assert
        age.Should().BeGreaterThan(18);
        age.Should().BeGreaterThanOrEqualTo(25);
        age.Should().BeLessThan(100);
        age.Should().BeLessThanOrEqualTo(25);
    }

    [TestMethod]
    public void Value_MeetsMinimumThreshold()
    {
        // Arrange
        var score = 85;

        // Act & Assert
        score.Should().BeGreaterThanOrEqualTo(70, "passing score is 70");
    }
}
```

## Sign and Special Value Assertions

```csharp
[TestClass]
public class SignTests
{
    [TestMethod]
    public void Value_IsPositive()
    {
        // Arrange
        var profit = 1000m;

        // Act & Assert
        profit.Should().BePositive();
        profit.Should().NotBeNegative();
    }

    [TestMethod]
    public void Value_IsNegative()
    {
        // Arrange
        var loss = -500m;

        // Act & Assert
        loss.Should().BeNegative();
        loss.Should().NotBePositive();
    }

    [TestMethod]
    public void Value_IsZero()
    {
        // Arrange
        var balance = 0m;

        // Act & Assert
        balance.Should().Be(0m);
        balance.Should().NotBePositive();
        balance.Should().NotBeNegative();
    }
}
```

## Floating-Point Precision Testing

```csharp
[TestClass]
public class FloatingPointTests
{
    [TestMethod]
    public void FloatingPoint_ApproximateEquality()
    {
        // Arrange
        var result = 0.1 + 0.2;
        var expected = 0.3;

        // Act & Assert
        result.Should().BeApproximately(expected, precision: 0.0001);
    }

    [TestMethod]
    public void FloatingPoint_WithinTolerance()
    {
        // Arrange
        var measured = 99.7;
        var target = 100.0;

        // Act & Assert
        measured.Should().BeApproximately(target, precision: 0.5);
    }

    [TestMethod]
    public void Calculation_ProducesExpectedPrecision()
    {
        // Arrange
        var value = 1.0 / 3.0 * 3.0;

        // Act & Assert
        value.Should().BeApproximately(1.0, precision: 0.000001);
    }
}
```

## Domain-Specific Numeric Testing

```csharp
public record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    private Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    public static Result<Money> Create(decimal amount, string currency)
    {
        if (amount < 0)
            return Result.Failure<Money>("Amount cannot be negative");

        if (string.IsNullOrWhiteSpace(currency))
            return Result.Failure<Money>("Currency is required");

        return Result.Success(new Money(amount, currency));
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");

        return new Money(Amount + other.Amount, Currency);
    }
}

[TestClass]
public class MoneyNumericTests
{
    [TestMethod]
    public void Money_Create_WithPositiveAmount_Succeeds()
    {
        // Arrange
        var amount = 100.50m;

        // Act
        var result = Money.Create(amount, "USD");

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Amount.Should().Be(100.50m);
        result.Value.Amount.Should().BePositive();
    }

    [TestMethod]
    public void Money_Create_WithNegativeAmount_Fails()
    {
        // Arrange
        var amount = -50m;

        // Act
        var result = Money.Create(amount, "USD");

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("negative");
    }

    [TestMethod]
    public void Money_Add_ReturnsCorrectSum()
    {
        // Arrange
        var money1 = Money.Create(100m, "USD").Value;
        var money2 = Money.Create(50m, "USD").Value;

        // Act
        var sum = money1.Add(money2);

        // Assert
        sum.Amount.Should().Be(150m);
        sum.Amount.Should().BeGreaterThan(money1.Amount);
        sum.Amount.Should().BeGreaterThan(money2.Amount);
    }
}
```

## Percentage and Ratio Testing

```csharp
[TestClass]
public class PercentageTests
{
    [TestMethod]
    public void Percentage_IsInValidRange()
    {
        // Arrange
        var percentage = 0.15m; // 15%

        // Act & Assert
        percentage.Should().BeInRange(0m, 1m);
        percentage.Should().BeGreaterThanOrEqualTo(0m);
        percentage.Should().BeLessThanOrEqualTo(1m);
    }

    [TestMethod]
    public void ConvertToPercentage_ReturnsCorrectValue()
    {
        // Arrange
        var fraction = 0.25m;

        // Act
        var percentage = fraction * 100m;

        // Assert
        percentage.Should().Be(25m);
        percentage.Should().BeInRange(0m, 100m);
    }
}
```

## Arithmetic Invariant Testing

```csharp
[TestClass]
public class ArithmeticInvariantTests
{
    [TestMethod]
    public void Addition_IsCommutative()
    {
        // Arrange
        var a = 10;
        var b = 20;

        // Act
        var sum1 = a + b;
        var sum2 = b + a;

        // Assert
        sum1.Should().Be(sum2);
    }

    [TestMethod]
    public void Subtraction_InvertsAddition()
    {
        // Arrange
        var original = 100;
        var amount = 25;

        // Act
        var result = original + amount - amount;

        // Assert
        result.Should().Be(original);
    }

    [TestMethod]
    public void Multiplication_ByZero_ReturnsZero()
    {
        // Arrange
        var value = 42;

        // Act
        var result = value * 0;

        // Assert
        result.Should().Be(0);
    }
}
```

## Why It's Important

1. **Precision**: Handle floating-point comparison correctly
2. **Clear Intent**: Express numeric constraints explicitly
3. **Better Errors**: Show expected vs actual with clear messages
4. **Range Safety**: Test boundary conditions easily
5. **Domain Rules**: Verify business constraints on numbers

## Guidelines

**Common Numeric Assertions**
- `.Should().Be(expected)` - exact equality
- `.Should().BeApproximately(expected, precision)` - floating-point equality
- `.Should().BeInRange(min, max)` - range check
- `.Should().BePositive()` / `.BeNegative()` - sign check
- `.Should().BeGreaterThan(value)` - comparison
- `.Should().BeLessThanOrEqualTo(value)` - comparison

**Floating-Point Numbers**
- Always use `.BeApproximately()` for double/float
- Specify appropriate precision for your domain
- Avoid exact equality checks for calculated floating-point values

**Decimal Numbers**
- Can use exact equality (`.Be()`)
- Appropriate for money and financial calculations
- No precision loss in decimal arithmetic

## Common Pitfalls

❌ **Exact equality for floating-point**
```csharp
// Unreliable due to floating-point precision
(0.1 + 0.2).Should().Be(0.3);
```

✅ **Use approximate equality**
```csharp
// Reliable with tolerance
(0.1 + 0.2).Should().BeApproximately(0.3, precision: 0.0001);
```

❌ **Too strict precision**
```csharp
// May fail due to rounding
calculated.Should().BeApproximately(expected, precision: 0.0000000001);
```

✅ **Appropriate precision for domain**
```csharp
// Reasonable for most calculations
calculated.Should().BeApproximately(expected, precision: 0.0001);
```

## See Also

- [Testing Value Objects](./testing-value-objects.md) — numeric value objects
- [Testing Domain Invariants](./testing-domain-invariants.md) — numeric constraints
- [Units of Measure](../patterns/units-of-measure.md) — type-safe quantities
- [Refinement Types](../patterns/refinement-types.md) — constrained numbers
