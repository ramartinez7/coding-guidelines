# Testing Equality and Comparison

> Verify equality operators, IEquatable<T>, and comparison logic—ensure consistent behavior across all equality methods.

## Problem

Types that implement equality or comparison must ensure consistent behavior across `Equals()`, `==`, `!=`, `GetHashCode()`, and comparison operators. Missing or incorrect implementations lead to subtle bugs.

## Pattern

Test all equality and comparison methods together to ensure consistency. Verify reflexive, symmetric, and transitive properties of equality.

## Example

### ❌ Before - Incomplete Equality Testing

```csharp
[TestMethod]
public void Money_Equality_Works()
{
    var money1 = Money.USD(100m);
    var money2 = Money.USD(100m);
    Assert.AreEqual(money1, money2);
}
// ❌ Missing: operator==, operator!=, GetHashCode, null handling
```

### ✅ After - Complete Equality Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class MoneyEqualityTests
{
    [TestMethod]
    public void Equals_WithEqualValues_ReturnsTrue()
    {
        // Arrange
        var money1 = Money.USD(100m);
        var money2 = Money.USD(100m);

        // Act
        var result = money1.Equals(money2);

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void EqualityOperator_WithEqualValues_ReturnsTrue()
    {
        // Arrange
        var money1 = Money.USD(100m);
        var money2 = Money.USD(100m);

        // Act
        var result = money1 == money2;

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void InequalityOperator_WithEqualValues_ReturnsFalse()
    {
        // Arrange
        var money1 = Money.USD(100m);
        var money2 = Money.USD(100m);

        // Act
        var result = money1 != money2;

        // Assert
        result.Should().BeFalse();
    }

    [TestMethod]
    public void GetHashCode_WithEqualValues_ReturnsSameHashCode()
    {
        // Arrange
        var money1 = Money.USD(100m);
        var money2 = Money.USD(100m);

        // Act
        var hash1 = money1.GetHashCode();
        var hash2 = money2.GetHashCode();

        // Assert
        hash1.Should().Be(hash2);
    }

    [TestMethod]
    public void Equals_WithNull_ReturnsFalse()
    {
        // Arrange
        var money = Money.USD(100m);

        // Act
        var result = money.Equals(null);

        // Assert
        result.Should().BeFalse();
    }

    [TestMethod]
    public void Equals_WithDifferentType_ReturnsFalse()
    {
        // Arrange
        var money = Money.USD(100m);
        var notMoney = "100 USD";

        // Act
        var result = money.Equals(notMoney);

        // Assert
        result.Should().BeFalse();
    }

    [TestMethod]
    public void Equals_WithSameInstance_ReturnsTrue()
    {
        // Arrange
        var money = Money.USD(100m);

        // Act
        var result = money.Equals(money);

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void Equals_IsSymmetric()
    {
        // Arrange
        var money1 = Money.USD(100m);
        var money2 = Money.USD(100m);

        // Act & Assert
        money1.Equals(money2).Should().Be(money2.Equals(money1));
    }
}
```

## Testing Comparison Operations

```csharp
public record Temperature : IComparable<Temperature>
{
    public decimal Celsius { get; }

    private Temperature(decimal celsius) => Celsius = celsius;

    public static Temperature FromCelsius(decimal celsius) => new(celsius);

    public int CompareTo(Temperature? other)
    {
        if (other is null) return 1;
        return Celsius.CompareTo(other.Celsius);
    }

    public static bool operator <(Temperature left, Temperature right) => left.CompareTo(right) < 0;
    public static bool operator >(Temperature left, Temperature right) => left.CompareTo(right) > 0;
    public static bool operator <=(Temperature left, Temperature right) => left.CompareTo(right) <= 0;
    public static bool operator >=(Temperature left, Temperature right) => left.CompareTo(right) >= 0;
}

[TestClass]
public class TemperatureComparisonTests
{
    [TestMethod]
    public void CompareTo_WithGreaterValue_ReturnsPositive()
    {
        // Arrange
        var temp1 = Temperature.FromCelsius(25m);
        var temp2 = Temperature.FromCelsius(20m);

        // Act
        var result = temp1.CompareTo(temp2);

        // Assert
        result.Should().BePositive();
    }

    [TestMethod]
    public void CompareTo_WithLesserValue_ReturnsNegative()
    {
        // Arrange
        var temp1 = Temperature.FromCelsius(20m);
        var temp2 = Temperature.FromCelsius(25m);

        // Act
        var result = temp1.CompareTo(temp2);

        // Assert
        result.Should().BeNegative();
    }

    [TestMethod]
    public void CompareTo_WithEqualValue_ReturnsZero()
    {
        // Arrange
        var temp1 = Temperature.FromCelsius(20m);
        var temp2 = Temperature.FromCelsius(20m);

        // Act
        var result = temp1.CompareTo(temp2);

        // Assert
        result.Should().Be(0);
    }

    [TestMethod]
    public void GreaterThanOperator_WithGreaterValue_ReturnsTrue()
    {
        // Arrange
        var temp1 = Temperature.FromCelsius(25m);
        var temp2 = Temperature.FromCelsius(20m);

        // Act
        var result = temp1 > temp2;

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void LessThanOperator_WithLesserValue_ReturnsTrue()
    {
        // Arrange
        var temp1 = Temperature.FromCelsius(20m);
        var temp2 = Temperature.FromCelsius(25m);

        // Act
        var result = temp1 < temp2;

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void GreaterThanOrEqual_WithEqualValues_ReturnsTrue()
    {
        // Arrange
        var temp1 = Temperature.FromCelsius(20m);
        var temp2 = Temperature.FromCelsius(20m);

        // Act
        var result = temp1 >= temp2;

        // Assert
        result.Should().BeTrue();
    }

    [TestMethod]
    public void LessThanOrEqual_WithEqualValues_ReturnsTrue()
    {
        // Arrange
        var temp1 = Temperature.FromCelsius(20m);
        var temp2 = Temperature.FromCelsius(20m);

        // Act
        var result = temp1 <= temp2;

        // Assert
        result.Should().BeTrue();
    }
}
```

## Testing with FluentAssertions Extensions

```csharp
[TestClass]
public class MoneyFluentTests
{
    [TestMethod]
    public void Money_ShouldBeEquivalentTo_OtherMoney()
    {
        // Arrange
        var money1 = Money.USD(100m);
        var money2 = Money.USD(100m);

        // Act & Assert
        money1.Should().BeEquivalentTo(money2);
    }

    [TestMethod]
    public void Money_InCollection_CanBeTested()
    {
        // Arrange
        var money = Money.USD(100m);
        var collection = new List<Money> { Money.USD(50m), Money.USD(100m), Money.USD(150m) };

        // Act & Assert
        collection.Should().Contain(money);
    }
}
```

## Why It's Important

1. **Consistency**: All equality methods must agree
2. **Hash Collections**: GetHashCode must be consistent with Equals for dictionaries/sets
3. **Symmetry**: If A equals B, then B must equal A
4. **Transitivity**: If A equals B and B equals C, then A must equal C
5. **Comparison Contracts**: Comparison operators must be consistent with CompareTo

## Guidelines

**What to Test for Equality**
- `Equals()` method with equal and unequal values
- `operator==` and `operator!=`
- `GetHashCode()` consistency
- Null handling
- Different type handling
- Reflexivity, symmetry, and transitivity

**What to Test for Comparison**
- `CompareTo()` with greater, lesser, and equal values
- All comparison operators: `<`, `>`, `<=`, `>=`
- Consistency between operators and CompareTo
- Null handling in CompareTo

**FluentAssertions for Equality**
- `.Should().Be()` - value equality
- `.Should().BeEquivalentTo()` - deep equality
- `.Should().BeSameAs()` - reference equality
- `.Should().BePositive()` / `.BeNegative()` / `.Be(0)` - comparison results

## Common Pitfalls

❌ **Testing only Equals, not operators**
```csharp
// Incomplete
money1.Equals(money2).Should().BeTrue();
```

✅ **Test all equality methods**
```csharp
// Complete
money1.Equals(money2).Should().BeTrue();
(money1 == money2).Should().BeTrue();
(money1 != money2).Should().BeFalse();
money1.GetHashCode().Should().Be(money2.GetHashCode());
```

❌ **Forgetting hash code consistency**
```csharp
// Missing hash code test
money1.Should().Be(money2);
```

✅ **Always test hash codes**
```csharp
// Complete
money1.Should().Be(money2);
money1.GetHashCode().Should().Be(money2.GetHashCode());
```

## See Also

- [Testing Value Objects](./testing-value-objects.md) — complete value object testing
- [Testing Record Types](./testing-record-types.md) — record equality
- [Value Semantics](../patterns/value-semantics.md) — equality philosophy
- [Record Equality](../patterns/record-equality.md) — record equality implementation
