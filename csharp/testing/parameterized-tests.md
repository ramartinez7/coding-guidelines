# Parameterized Tests

> Run the same test logic with different inputs—reduce duplication while covering more scenarios.

## Problem

Similar tests with different inputs lead to duplication. Adding new test cases requires copying entire test methods.

## Pattern

Use parameterized tests (also called data-driven tests) to run the same test logic with multiple inputs.

## Example

### ❌ Before - Duplicated Tests

```csharp
[Fact]
public void CreateEmail_WithNull_ReturnsFailure()
{
    var result = EmailAddress.Create(null);
    Assert.True(result.IsFailure);
}

[Fact]
public void CreateEmail_WithEmpty_ReturnsFailure()
{
    var result = EmailAddress.Create("");
    Assert.True(result.IsFailure);
}

[Fact]
public void CreateEmail_WithWhitespace_ReturnsFailure()
{
    var result = EmailAddress.Create("   ");
    Assert.True(result.IsFailure);
}

[Fact]
public void CreateEmail_WithoutAtSign_ReturnsFailure()
{
    var result = EmailAddress.Create("invalid");
    Assert.True(result.IsFailure);
}
```

### ✅ After - Parameterized Test

```csharp
[Theory]
[InlineData(null)]
[InlineData("")]
[InlineData("   ")]
[InlineData("invalid")]
[InlineData("@example.com")]
[InlineData("user@")]
[InlineData("user@@example.com")]
public void CreateEmail_WithInvalidInput_ReturnsFailure(string invalidInput)
{
    // Arrange & Act
    var result = EmailAddress.Create(invalidInput);

    // Assert
    Assert.True(result.IsFailure);
}
```

## xUnit Theory Patterns

### InlineData - Simple Values

```csharp
[Theory]
[InlineData(0, 100, 100)]
[InlineData(100, 0, 100)]
[InlineData(50, 50, 100)]
[InlineData(75, 25, 100)]
public void Add_TwoMoneyAmounts_ReturnsSum(decimal amount1, decimal amount2, decimal expected)
{
    // Arrange
    var money1 = Money.USD(amount1);
    var money2 = Money.USD(amount2);

    // Act
    var result = money1.Add(money2);

    // Assert
    Assert.Equal(Money.USD(expected), result);
}
```

### MemberData - Complex Objects

```csharp
public class MoneyTests
{
    public static IEnumerable<object[]> ValidMoneyAmounts =>
        new List<object[]>
        {
            new object[] { 0m, "USD", "$0.00" },
            new object[] { 100m, "USD", "$100.00" },
            new object[] { 99.99m, "USD", "$99.99" },
            new object[] { 1000.50m, "EUR", "€1,000.50" },
        };

    [Theory]
    [MemberData(nameof(ValidMoneyAmounts))]
    public void ToString_WithAmount_FormatsCorrectly(decimal amount, string currency, string expected)
    {
        // Arrange
        var money = Money.Create(amount, Currency.FromCode(currency));

        // Act
        var result = money.ToString();

        // Assert
        Assert.Equal(expected, result);
    }
}
```

### ClassData - Encapsulated Test Data

```csharp
public class ValidEmailData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { "user@example.com" };
        yield return new object[] { "user+tag@example.com" };
        yield return new object[] { "user.name@example.com" };
        yield return new object[] { "user@sub.example.com" };
        yield return new object[] { "123@example.com" };
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(ValidEmailData))]
public void CreateEmail_WithValidFormat_ReturnsSuccess(string validEmail)
{
    // Arrange & Act
    var result = EmailAddress.Create(validEmail);

    // Assert
    Assert.True(result.IsSuccess);
    Assert.Equal(validEmail.ToLowerInvariant(), result.Value.ToString());
}
```

## Testing Value Objects

```csharp
public class AgeTests
{
    [Theory]
    [InlineData(-1)]
    [InlineData(-100)]
    [InlineData(151)]
    [InlineData(1000)]
    public void Create_WithInvalidAge_ReturnsFailure(int invalidAge)
    {
        var result = Age.Create(invalidAge);
        Assert.True(result.IsFailure);
    }

    [Theory]
    [InlineData(0)]
    [InlineData(1)]
    [InlineData(18)]
    [InlineData(65)]
    [InlineData(150)]
    public void Create_WithValidAge_ReturnsSuccess(int validAge)
    {
        var result = Age.Create(validAge);
        Assert.True(result.IsSuccess);
        Assert.Equal(validAge, result.Value.Value);
    }
}
```

## Testing State Transitions

```csharp
public class OrderStatusTests
{
    public static IEnumerable<object[]> ValidTransitions =>
        new List<object[]>
        {
            new object[] { OrderStatus.Pending, OrderStatus.Confirmed },
            new object[] { OrderStatus.Confirmed, OrderStatus.Processing },
            new object[] { OrderStatus.Processing, OrderStatus.Shipped },
            new object[] { OrderStatus.Shipped, OrderStatus.Delivered },
            new object[] { OrderStatus.Pending, OrderStatus.Cancelled },
            new object[] { OrderStatus.Confirmed, OrderStatus.Cancelled },
        };

    public static IEnumerable<object[]> InvalidTransitions =>
        new List<object[]>
        {
            new object[] { OrderStatus.Delivered, OrderStatus.Pending },
            new object[] { OrderStatus.Cancelled, OrderStatus.Processing },
            new object[] { OrderStatus.Shipped, OrderStatus.Confirmed },
        };

    [Theory]
    [MemberData(nameof(ValidTransitions))]
    public void TransitionTo_ValidTransition_Succeeds(OrderStatus from, OrderStatus to)
    {
        // Arrange
        var order = OrderMother.OrderInStatus(from);

        // Act
        var result = order.TransitionTo(to);

        // Assert
        Assert.True(result.IsSuccess);
        Assert.Equal(to, order.Status);
    }

    [Theory]
    [MemberData(nameof(InvalidTransitions))]
    public void TransitionTo_InvalidTransition_ReturnsFailure(OrderStatus from, OrderStatus to)
    {
        // Arrange
        var order = OrderMother.OrderInStatus(from);

        // Act
        var result = order.TransitionTo(to);

        // Assert
        Assert.True(result.IsFailure);
        Assert.Equal(from, order.Status); // Status unchanged
    }
}
```

## Testing Calculations

```csharp
public class DiscountTests
{
    public static IEnumerable<object[]> DiscountScenarios =>
        new List<object[]>
        {
            new object[] { 100m, 0.10m, 10m },    // 10% of $100
            new object[] { 100m, 0.20m, 20m },    // 20% of $100
            new object[] { 50m, 0.15m, 7.50m },   // 15% of $50
            new object[] { 200m, 0.25m, 50m },    // 25% of $200
            new object[] { 99.99m, 0.10m, 10m },  // 10% of $99.99 (rounded)
        };

    [Theory]
    [MemberData(nameof(DiscountScenarios))]
    public void CalculateDiscount_WithPercentage_ReturnsCorrectAmount(
        decimal subtotal, 
        decimal percentage, 
        decimal expectedDiscount)
    {
        // Arrange
        var order = OrderBuilder.Default()
            .WithSubtotal(Money.USD(subtotal))
            .Build();
        var discount = Percentage.FromDecimal(percentage);

        // Act
        var result = DiscountCalculator.Calculate(order, discount);

        // Assert
        Assert.Equal(Money.USD(expectedDiscount), result);
    }
}
```

## Composite Test Cases

```csharp
public class OrderValidationTests
{
    public record TestCase(
        string Description,
        Order Order,
        bool ExpectedValid,
        string? ExpectedError = null
    );

    public static IEnumerable<object[]> ValidationCases =>
        new List<object[]>
        {
            new object[] { new TestCase(
                "Valid order with items",
                OrderBuilder.Default().WithThreeItems().Build(),
                ExpectedValid: true
            )},
            new object[] { new TestCase(
                "Empty order",
                OrderBuilder.Default().WithEmptyCart().Build(),
                ExpectedValid: false,
                ExpectedError: "Order must contain at least one item"
            )},
            new object[] { new TestCase(
                "Order with negative total",
                OrderBuilder.Default().WithNegativeTotal().Build(),
                ExpectedValid: false,
                ExpectedError: "Total cannot be negative"
            )},
        };

    [Theory]
    [MemberData(nameof(ValidationCases))]
    public void Validate_Order_ReturnsExpectedResult(TestCase testCase)
    {
        // Act
        var result = testCase.Order.Validate();

        // Assert
        Assert.Equal(testCase.ExpectedValid, result.IsSuccess);
        
        if (!testCase.ExpectedValid)
        {
            Assert.Contains(testCase.ExpectedError!, result.Error);
        }
    }
}
```

## Custom Attributes

```csharp
public class JsonFileDataAttribute : DataAttribute
{
    private readonly string filePath;

    public JsonFileDataAttribute(string filePath)
    {
        this.filePath = filePath;
    }

    public override IEnumerable<object[]> GetData(MethodInfo testMethod)
    {
        var json = File.ReadAllText(filePath);
        var testCases = JsonSerializer.Deserialize<List<TestCase>>(json);
        
        return testCases.Select(tc => new object[] { tc });
    }
}

[Theory]
[JsonFileData("testdata/email-validations.json")]
public void CreateEmail_WithTestData_MatchesExpectation(EmailTestCase testCase)
{
    var result = EmailAddress.Create(testCase.Input);
    Assert.Equal(testCase.ShouldSucceed, result.IsSuccess);
}
```

## Benefits

1. **Less code**: One test method covers many scenarios
2. **Easy to add cases**: Just add data, no code duplication
3. **Clear intent**: Data shows what's being tested
4. **Maintainability**: Logic changes in one place
5. **Coverage**: Easier to test edge cases

## Guidelines

- Use `InlineData` for simple primitive values
- Use `MemberData` for complex objects or many cases
- Use `ClassData` when test data needs setup logic
- Give each test case a meaningful description
- Keep test logic simple—complexity should be in data, not code

## See Also

- [Property-Based Testing](./property-based-testing.md) — testing with generated inputs
- [Testing Domain Invariants](./testing-domain-invariants.md) — validating business rules
- [Testing Value Objects](./testing-value-objects.md) — equality and validation
- [One Assertion Per Test](./one-assertion-per-test.md) — focused tests
