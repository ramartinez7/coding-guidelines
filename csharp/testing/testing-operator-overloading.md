# Testing Operator Overloading

> Test custom operators—verify arithmetic, comparison, equality, and conversion operators work correctly.

## Problem

Custom operators enable natural syntax for domain types. Tests must verify operators maintain mathematical properties and handle edge cases.

## Example

```csharp
public record Money(decimal Amount, string Currency)
{
    public static Money operator +(Money left, Money right)
    {
        if (left.Currency != right.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return new Money(left.Amount + right.Amount, left.Currency);
    }
}

[Fact]
public void AddOperator_SameCurrency_SumsAmounts()
{
    var money1 = new Money(10m, "USD");
    var money2 = new Money(20m, "USD");
    
    var result = money1 + money2;
    
    result.Should().Be(new Money(30m, "USD"));
}

[Fact]
public void AddOperator_DifferentCurrency_Throws()
{
    var usd = new Money(10m, "USD");
    var eur = new Money(20m, "EUR");
    
    Action act = () => { var _ = usd + eur; };
    
    act.Should().Throw<InvalidOperationException>();
}
```

## Testing Comparison Operators

```csharp
[Theory]
[InlineData(10, 20, true)]
[InlineData(20, 10, false)]
[InlineData(10, 10, false)]
public void LessThanOperator_VariousValues_ComparesCorrectly(
    decimal amount1, decimal amount2, bool expected)
{
    var money1 = new Money(amount1, "USD");
    var money2 = new Money(amount2, "USD");
    
    var result = money1 < money2;
    
    result.Should().Be(expected);
}
```

## See Also

- [Testing Equality Comparison](./testing-equality-comparison.md)
- [Testing Value Objects](./testing-value-objects.md)
