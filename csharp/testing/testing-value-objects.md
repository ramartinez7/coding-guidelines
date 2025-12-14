# Testing Value Objects

> Test value semantics—equality, immutability, and validation in domain value types.

## Problem

Value objects have specific semantics that must be tested: equality by value, immutability, validation, and proper hash codes. Missing these tests can lead to subtle bugs.

## Pattern

Test that value objects follow value semantics: structural equality, immutability, proper hash code implementation, and validation rules.

## Example

### ❌ Before - Incomplete Value Object Tests

```csharp
[Fact]
public void CreateMoney_Works()
{
    var money = Money.USD(100m);
    Assert.NotNull(money);
}
// ❌ Missing: equality, hash code, immutability, comparison
```

### ✅ After - Complete Value Object Tests

```csharp
[Fact]
public void Money_WithSameValues_AreEqual()
{
    var money1 = Money.USD(100m);
    var money2 = Money.USD(100m);
    
    Assert.Equal(money1, money2);
    Assert.True(money1 == money2);
}

[Fact]
public void Money_WithSameValues_HaveSameHashCode()
{
    var money1 = Money.USD(100m);
    var money2 = Money.USD(100m);
    
    Assert.Equal(money1.GetHashCode(), money2.GetHashCode());
}

[Fact]
public void Money_Operations_DoNotMutate()
{
    var original = Money.USD(100m);
    var added = original.Add(Money.USD(50m));
    
    Assert.Equal(Money.USD(100m), original); // Unchanged
    Assert.Equal(Money.USD(150m), added);
}
```

## Testing Equality

### Value Equality

```csharp
public class EmailAddressTests
{
    [Fact]
    public void Equality_WithSameValue_ReturnsTrue()
    {
        var email1 = EmailAddress.Create("user@example.com").Value;
        var email2 = EmailAddress.Create("user@example.com").Value;
        
        Assert.Equal(email1, email2);
        Assert.True(email1 == email2);
        Assert.False(email1 != email2);
    }

    [Fact]
    public void Equality_WithDifferentValues_ReturnsFalse()
    {
        var email1 = EmailAddress.Create("user1@example.com").Value;
        var email2 = EmailAddress.Create("user2@example.com").Value;
        
        Assert.NotEqual(email1, email2);
        Assert.False(email1 == email2);
        Assert.True(email1 != email2);
    }

    [Fact]
    public void Equality_IsCaseInsensitive()
    {
        // Email addresses should be case-insensitive
        var email1 = EmailAddress.Create("User@Example.COM").Value;
        var email2 = EmailAddress.Create("user@example.com").Value;
        
        Assert.Equal(email1, email2);
    }

    [Fact]
    public void Equals_WithNull_ReturnsFalse()
    {
        var email = EmailAddress.Create("user@example.com").Value;
        
        Assert.False(email.Equals(null));
        Assert.False(email == null);
    }

    [Fact]
    public void Equals_WithSelf_ReturnsTrue()
    {
        var email = EmailAddress.Create("user@example.com").Value;
        
        Assert.True(email.Equals(email));
        Assert.True(email == email);
    }
}
```

### Record Equality

```csharp
public class MoneyTests
{
    [Fact]
    public void Record_Equality_WorksCorrectly()
    {
        // Using C# records for automatic value equality
        var money1 = new Money(100m, Currency.USD);
        var money2 = new Money(100m, Currency.USD);
        var money3 = new Money(100m, Currency.EUR);
        
        Assert.Equal(money1, money2);
        Assert.NotEqual(money1, money3);
    }

    [Fact]
    public void With_Expression_CreatesNewInstance()
    {
        var original = new Money(100m, Currency.USD);
        var modified = original with { Amount = 200m };
        
        Assert.Equal(Money.USD(100m), original); // Unchanged
        Assert.Equal(Money.USD(200m), modified);
    }
}
```

## Testing Hash Codes

```csharp
public class HashCodeTests
{
    [Fact]
    public void GetHashCode_ForEqualValues_ReturnsSameHash()
    {
        var money1 = Money.USD(100m);
        var money2 = Money.USD(100m);
        
        Assert.Equal(money1.GetHashCode(), money2.GetHashCode());
    }

    [Fact]
    public void GetHashCode_ForDifferentValues_ReturnsDifferentHashes()
    {
        var money1 = Money.USD(100m);
        var money2 = Money.USD(200m);
        
        // While not guaranteed, should usually differ
        Assert.NotEqual(money1.GetHashCode(), money2.GetHashCode());
    }

    [Fact]
    public void ValueObjects_CanBeUsedInHashSets()
    {
        var money1 = Money.USD(100m);
        var money2 = Money.USD(100m); // Same value
        var money3 = Money.USD(200m);
        
        var set = new HashSet<Money> { money1, money2, money3 };
        
        Assert.Equal(2, set.Count); // money1 and money2 are same
        Assert.Contains(Money.USD(100m), set);
        Assert.Contains(Money.USD(200m), set);
    }

    [Fact]
    public void ValueObjects_CanBeUsedAsDictionaryKeys()
    {
        var dict = new Dictionary<CustomerId, string>();
        var customerId = CustomerId.New();
        
        dict[customerId] = "John Doe";
        
        Assert.Equal("John Doe", dict[customerId]);
    }
}
```

## Testing Immutability

```csharp
public class ImmutabilityTests
{
    [Fact]
    public void Add_DoesNotMutateOriginal()
    {
        var original = Money.USD(100m);
        var originalValue = original.Amount;
        
        var result = original.Add(Money.USD(50m));
        
        Assert.Equal(originalValue, original.Amount); // Unchanged
        Assert.Equal(Money.USD(150m), result);
    }

    [Fact]
    public void Multiply_DoesNotMutateOriginal()
    {
        var original = Money.USD(100m);
        
        var doubled = original.Multiply(2m);
        
        Assert.Equal(Money.USD(100m), original);
        Assert.Equal(Money.USD(200m), doubled);
    }

    [Fact]
    public void ToString_AndBackToObject_PreservesValue()
    {
        var original = EmailAddress.Create("user@example.com").Value;
        var text = original.ToString();
        var parsed = EmailAddress.Create(text).Value;
        
        Assert.Equal(original, parsed);
    }
}
```

## Testing Validation

```csharp
public class ValidationTests
{
    [Theory]
    [InlineData("user@example.com")]
    [InlineData("test+tag@sub.example.com")]
    public void Create_WithValidEmail_Succeeds(string validEmail)
    {
        var result = EmailAddress.Create(validEmail);
        
        Assert.True(result.IsSuccess);
        Assert.Equal(validEmail.ToLowerInvariant(), result.Value.ToString());
    }

    [Theory]
    [InlineData(null)]
    [InlineData("")]
    [InlineData("invalid")]
    public void Create_WithInvalidEmail_ReturnsFailure(string invalidEmail)
    {
        var result = EmailAddress.Create(invalidEmail);
        
        Assert.True(result.IsFailure);
    }

    [Fact]
    public void Create_NormalizesValue()
    {
        var result = EmailAddress.Create("  USER@EXAMPLE.COM  ");
        
        Assert.True(result.IsSuccess);
        Assert.Equal("user@example.com", result.Value.ToString());
    }
}
```

## Testing Comparison

```csharp
public class ComparisonTests
{
    [Fact]
    public void CompareTo_OrdersCorrectly()
    {
        var money1 = Money.USD(100m);
        var money2 = Money.USD(200m);
        var money3 = Money.USD(100m);
        
        Assert.True(money1.CompareTo(money2) < 0);
        Assert.True(money2.CompareTo(money1) > 0);
        Assert.Equal(0, money1.CompareTo(money3));
    }

    [Fact]
    public void ComparisonOperators_WorkCorrectly()
    {
        var money1 = Money.USD(100m);
        var money2 = Money.USD(200m);
        
        Assert.True(money1 < money2);
        Assert.True(money2 > money1);
        Assert.True(money1 <= money2);
        Assert.True(money2 >= money1);
        Assert.True(money1 <= Money.USD(100m));
        Assert.True(money1 >= Money.USD(100m));
    }

    [Fact]
    public void Sorting_WorksCorrectly()
    {
        var amounts = new[]
        {
            Money.USD(300m),
            Money.USD(100m),
            Money.USD(200m),
        };
        
        var sorted = amounts.OrderBy(x => x).ToList();
        
        Assert.Equal(Money.USD(100m), sorted[0]);
        Assert.Equal(Money.USD(200m), sorted[1]);
        Assert.Equal(Money.USD(300m), sorted[2]);
    }
}
```

## Testing ToString and Formatting

```csharp
public class FormattingTests
{
    [Theory]
    [InlineData(0, "$0.00")]
    [InlineData(100, "$100.00")]
    [InlineData(99.99, "$99.99")]
    [InlineData(1000.50, "$1,000.50")]
    public void ToString_FormatsCorrectly(decimal amount, string expected)
    {
        var money = Money.USD(amount);
        
        Assert.Equal(expected, money.ToString());
    }

    [Fact]
    public void ToString_IncludesCurrency()
    {
        var usd = Money.USD(100m);
        var eur = Money.EUR(100m);
        
        Assert.Contains("$", usd.ToString());
        Assert.Contains("€", eur.ToString());
    }
}
```

## Testing Copy Semantics

```csharp
public class CopyTests
{
    [Fact]
    public void Assignment_CreatesReference_NotCopy()
    {
        // For reference types
        var original = EmailAddress.Create("user@example.com").Value;
        var reference = original;
        
        Assert.Same(original, reference);
    }

    [Fact]
    public void StructValueObject_Copies_OnAssignment()
    {
        // For value type structs
        var original = new Percentage(0.15m);
        var copy = original;
        
        // Both have same value but are different instances
        Assert.Equal(original, copy);
    }
}
```

## Testing Serialization

```csharp
public class SerializationTests
{
    [Fact]
    public void JsonSerialization_RoundTrips()
    {
        var original = Money.USD(99.99m);
        
        var json = JsonSerializer.Serialize(original);
        var deserialized = JsonSerializer.Deserialize<Money>(json);
        
        Assert.Equal(original, deserialized);
    }

    [Fact]
    public void JsonSerialization_UsesReadableFormat()
    {
        var money = Money.USD(100m);
        
        var json = JsonSerializer.Serialize(money);
        
        Assert.Contains("100", json);
        Assert.Contains("USD", json);
    }
}
```

## Testing Domain Operations

```csharp
public class DomainOperationsTests
{
    [Fact]
    public void Add_SameCurrency_Succeeds()
    {
        var money1 = Money.USD(100m);
        var money2 = Money.USD(50m);
        
        var result = money1.Add(money2);
        
        Assert.Equal(Money.USD(150m), result);
    }

    [Fact]
    public void Add_DifferentCurrency_ThrowsException()
    {
        var usd = Money.USD(100m);
        var eur = Money.EUR(50m);
        
        Assert.Throws<InvalidOperationException>(() =>
            usd.Add(eur));
    }

    [Fact]
    public void Multiply_ByFactor_CalculatesCorrectly()
    {
        var money = Money.USD(100m);
        
        var doubled = money.Multiply(2m);
        var halved = money.Multiply(0.5m);
        
        Assert.Equal(Money.USD(200m), doubled);
        Assert.Equal(Money.USD(50m), halved);
    }
}
```

## Testing Edge Cases

```csharp
public class EdgeCaseTests
{
    [Fact]
    public void Money_WithMaxValue_Handled()
    {
        var max = Money.USD(decimal.MaxValue);
        
        Assert.NotNull(max);
        Assert.Equal(decimal.MaxValue, max.Amount);
    }

    [Fact]
    public void Money_WithZero_IsValid()
    {
        var zero = Money.USD(0m);
        
        Assert.Equal(0m, zero.Amount);
    }

    [Fact]
    public void Email_WithUnicodeCharacters_HandledCorrectly()
    {
        var result = EmailAddress.Create("用户@例え.jp");
        
        // Depending on business rules
        Assert.True(result.IsSuccess || result.IsFailure);
    }
}
```

## Guidelines

1. **Test equality thoroughly**: Same values, different values, null
2. **Test hash codes**: Equal objects must have equal hash codes
3. **Verify immutability**: Operations create new instances
4. **Test validation**: Both valid and invalid inputs
5. **Test operations**: Domain-specific behaviors
6. **Use property-based testing** for value objects

## Benefits

1. **Correctness**: Ensures value semantics work properly
2. **Collections**: Can safely use in sets and dictionaries
3. **Confidence**: Know that equality and comparison work
4. **Documentation**: Tests show how value objects behave
5. **Regression prevention**: Catches breaking changes

## See Also

- [Testing Domain Invariants](./testing-domain-invariants.md) — validation rules
- [Property-Based Testing](./property-based-testing.md) — testing value object properties
- [Value Semantics](../patterns/value-semantics.md) — design pattern
- [Record Equality](../patterns/record-equality.md) — C# record types
