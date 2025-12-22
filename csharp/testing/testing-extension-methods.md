# Testing Extension Methods

> Test extension methods effectively—verify they enhance existing types without modifying them, handle edge cases, and compose correctly.

## Problem

Extension methods extend types you don't control. Tests must verify the extensions work correctly without breaking the extended type's invariants.

## Pattern

Test extension methods like static methods, focusing on inputs, outputs, and that they don't have unexpected side effects on the extended type.

## Example

### ❌ Before - Assuming Extension Behavior

```csharp
// No tests for extension method
public static class StringExtensions
{
    public static bool IsValidEmail(this string value)
    {
        return value.Contains("@");  // Oversimplified
    }
}

// Used without verification
var email = "user@example.com";
if (email.IsValidEmail()) { }
```

### ✅ After - Tested Extension Method

```csharp
public static class StringExtensions
{
    public static bool IsValidEmail(this string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return false;
            
        return value.Contains("@") && 
               value.IndexOf("@") > 0 &&
               value.IndexOf("@") < value.Length - 1;
    }
}

[Theory]
[InlineData("user@example.com", true)]
[InlineData("invalid", false)]
[InlineData("@example.com", false)]
[InlineData("user@", false)]
[InlineData("", false)]
[InlineData(null, false)]
public void IsValidEmail_VariousInputs_ValidatesCorrectly(string input, bool expected)
{
    // Act
    var result = input.IsValidEmail();

    // Assert
    result.Should().Be(expected);
}
```

## Testing Pure Extension Methods

### No Side Effects

```csharp
public static class MoneyExtensions
{
    public static Money AddTax(this Money amount, decimal taxRate)
    {
        var taxAmount = amount.Amount * taxRate;
        return Money.Create(amount.Amount + taxAmount, amount.Currency);
    }
}

[Fact]
public void AddTax_DoesNotModifyOriginal()
{
    // Arrange
    var original = Money.USD(100m);
    var originalAmount = original.Amount;

    // Act
    var withTax = original.AddTax(0.10m);

    // Assert
    original.Amount.Should().Be(originalAmount);  // Original unchanged
    withTax.Amount.Should().Be(110m);
}

[Theory]
[InlineData(100, 0.10, 110)]
[InlineData(50, 0.20, 60)]
[InlineData(0, 0.15, 0)]
public void AddTax_VariousAmounts_CalculatesCorrectly(
    decimal amount, 
    decimal taxRate, 
    decimal expected)
{
    // Arrange
    var money = Money.USD(amount);

    // Act
    var result = money.AddTax(taxRate);

    // Assert
    result.Amount.Should().Be(expected);
}
```

## Testing Null Handling

### Null Reference Extension Methods

```csharp
public static class StringExtensions
{
    public static string ToTitleCase(this string? value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return string.Empty;
            
        return CultureInfo.CurrentCulture.TextInfo.ToTitleCase(value.ToLower());
    }
}

[Theory]
[InlineData(null, "")]
[InlineData("", "")]
[InlineData("   ", "")]
[InlineData("hello world", "Hello World")]
[InlineData("HELLO WORLD", "Hello World")]
public void ToTitleCase_VariousInputs_ConvertsCorrectly(string? input, string expected)
{
    // Act
    var result = input.ToTitleCase();

    // Assert
    result.Should().Be(expected);
}
```

## Testing Collection Extensions

### LINQ-Style Extensions

```csharp
public static class EnumerableExtensions
{
    public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T?> source) 
        where T : class
    {
        return source.Where(x => x != null)!;
    }

    public static decimal SumMoney(this IEnumerable<Money> source)
    {
        return source.Sum(m => m.Amount);
    }
}

[Fact]
public void WhereNotNull_MixedNullsAndValues_FiltersNulls()
{
    // Arrange
    var items = new string?[] { "a", null, "b", null, "c" };

    // Act
    var result = items.WhereNotNull().ToList();

    // Assert
    result.Should().HaveCount(3);
    result.Should().Equal("a", "b", "c");
}

[Fact]
public void WhereNotNull_EmptySequence_ReturnsEmpty()
{
    // Arrange
    var items = Enumerable.Empty<string?>();

    // Act
    var result = items.WhereNotNull().ToList();

    // Assert
    result.Should().BeEmpty();
}

[Fact]
public void SumMoney_MultipleAmounts_SumsCorrectly()
{
    // Arrange
    var amounts = new[]
    {
        Money.USD(10m),
        Money.USD(20m),
        Money.USD(30m)
    };

    // Act
    var total = amounts.SumMoney();

    // Assert
    total.Should().Be(60m);
}
```

## Testing Fluent Extensions

### Method Chaining

```csharp
public static class OrderExtensions
{
    public static Order WithDiscount(this Order order, decimal percentage)
    {
        return order with 
        { 
            Total = order.Total.Multiply(1 - percentage) 
        };
    }

    public static Order WithShipping(this Order order, Money shippingCost)
    {
        return order with 
        { 
            Total = order.Total.Add(shippingCost) 
        };
    }
}

[Fact]
public void FluentExtensions_ChainedCalls_ApplyInOrder()
{
    // Arrange
    var order = Order.Create(CustomerId.New(), CreateItemsTotaling(Money.USD(100m)));

    // Act
    var result = order
        .WithDiscount(0.10m)      // 100 - 10% = 90
        .WithShipping(Money.USD(10m));  // 90 + 10 = 100

    // Assert
    result.Total.Should().Be(Money.USD(100m));
}

[Fact]
public void FluentExtensions_DoNotModifyIntermediate()
{
    // Arrange
    var order = Order.Create(CustomerId.New(), CreateItemsTotaling(Money.USD(100m)));

    // Act
    var discounted = order.WithDiscount(0.10m);
    var withShipping = discounted.WithShipping(Money.USD(10m));

    // Assert
    order.Total.Should().Be(Money.USD(100m));        // Original unchanged
    discounted.Total.Should().Be(Money.USD(90m));    // First step unchanged
    withShipping.Total.Should().Be(Money.USD(100m)); // Final result
}
```

## Testing Conversion Extensions

### Type Conversions

```csharp
public static class ConversionExtensions
{
    public static Result<EmailAddress> ToEmailAddress(this string value)
    {
        return EmailAddress.Create(value);
    }

    public static Money ToUSD(this decimal amount)
    {
        return Money.USD(amount);
    }

    public static CustomerId ToCustomerId(this Guid value)
    {
        return CustomerId.From(value);
    }
}

[Theory]
[InlineData("user@example.com", true)]
[InlineData("invalid", false)]
public void ToEmailAddress_VariousStrings_ConvertsOrFails(string input, bool shouldSucceed)
{
    // Act
    var result = input.ToEmailAddress();

    // Assert
    result.IsSuccess.Should().Be(shouldSucceed);
}

[Fact]
public void ToUSD_DecimalAmount_CreatesMoneyObject()
{
    // Arrange
    var amount = 123.45m;

    // Act
    var money = amount.ToUSD();

    // Assert
    money.Amount.Should().Be(123.45m);
    money.Currency.Should().Be("USD");
}
```

## Testing Validation Extensions

### Guard Clause Extensions

```csharp
public static class GuardExtensions
{
    public static T ThrowIfNull<T>(this T? value, string paramName) 
        where T : class
    {
        if (value == null)
            throw new ArgumentNullException(paramName);
            
        return value;
    }

    public static string ThrowIfNullOrEmpty(this string? value, string paramName)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Value cannot be null or empty", paramName);
            
        return value;
    }
}

[Fact]
public void ThrowIfNull_NullValue_ThrowsArgumentNullException()
{
    // Arrange
    string? value = null;

    // Act
    Action act = () => value.ThrowIfNull(nameof(value));

    // Assert
    act.Should().Throw<ArgumentNullException>()
        .WithParameterName(nameof(value));
}

[Fact]
public void ThrowIfNull_NonNullValue_ReturnsValue()
{
    // Arrange
    var value = "test";

    // Act
    var result = value.ThrowIfNull(nameof(value));

    // Assert
    result.Should().Be("test");
}

[Theory]
[InlineData(null)]
[InlineData("")]
[InlineData("   ")]
public void ThrowIfNullOrEmpty_InvalidValue_Throws(string? value)
{
    // Act
    Action act = () => value.ThrowIfNullOrEmpty(nameof(value));

    // Assert
    act.Should().Throw<ArgumentException>();
}
```

## Testing Async Extensions

### Async Extension Methods

```csharp
public static class TaskExtensions
{
    public static async Task<Result<T>> ToResult<T>(this Task<T> task)
    {
        try
        {
            var value = await task;
            return Result.Success(value);
        }
        catch (Exception ex)
        {
            return Result.Failure<T>(ex.Message);
        }
    }

    public static async Task<T> WithTimeout<T>(this Task<T> task, TimeSpan timeout)
    {
        var completedTask = await Task.WhenAny(task, Task.Delay(timeout));
        
        if (completedTask != task)
            throw new TimeoutException();
            
        return await task;
    }
}

[Fact]
public async Task ToResult_SuccessfulTask_ReturnsSuccess()
{
    // Arrange
    var task = Task.FromResult(42);

    // Act
    var result = await task.ToResult();

    // Assert
    result.Should().BeSuccess();
    result.Value.Should().Be(42);
}

[Fact]
public async Task ToResult_FailedTask_ReturnsFailure()
{
    // Arrange
    var task = Task.FromException<int>(new InvalidOperationException("Error"));

    // Act
    var result = await task.ToResult();

    // Assert
    result.Should().BeFailure();
    result.Error.Should().Contain("Error");
}

[Fact]
public async Task WithTimeout_CompletesInTime_ReturnsResult()
{
    // Arrange
    var task = Task.FromResult(42);

    // Act
    var result = await task.WithTimeout(TimeSpan.FromSeconds(1));

    // Assert
    result.Should().Be(42);
}

[Fact]
public async Task WithTimeout_ExceedsTimeout_ThrowsTimeoutException()
{
    // Arrange
    var task = Task.Delay(TimeSpan.FromSeconds(10)).ContinueWith(_ => 42);

    // Act
    Func<Task> act = async () => await task.WithTimeout(TimeSpan.FromMilliseconds(100));

    // Assert
    await act.Should().ThrowAsync<TimeoutException>();
}
```

## Testing Extension Method Composition

### Multiple Extensions Together

```csharp
public static class StringExtensions
{
    public static string RemoveWhitespace(this string value) =>
        new string(value.Where(c => !char.IsWhiteSpace(c)).ToArray());

    public static string Truncate(this string value, int maxLength) =>
        value.Length <= maxLength ? value : value[..maxLength];

    public static string WrapInQuotes(this string value) =>
        $"\"{value}\"";
}

[Fact]
public void ComposedExtensions_ChainedOperations_ApplyInOrder()
{
    // Arrange
    var input = "  Hello World  ";

    // Act
    var result = input
        .RemoveWhitespace()
        .Truncate(8)
        .WrapInQuotes();

    // Assert
    result.Should().Be("\"HelloWor\"");
}
```

## Guidelines

1. **Test null handling**: Extension methods on nullable types must handle nulls
2. **Test immutability**: Verify extensions don't modify original values
3. **Test edge cases**: Empty collections, zero values, boundary conditions
4. **Test composition**: Verify multiple extensions chain correctly
5. **Document behavior**: Clear test names explain what extension does

## Benefits

1. **Safe extensions**: Verified behavior before use
2. **Edge case coverage**: Tests catch null, empty, and boundary cases
3. **Composition confidence**: Know extensions work together correctly
4. **Maintainability**: Changes to extensions won't break unexpectedly
5. **Documentation**: Tests serve as usage examples

## See Also

- [Testing Guard Clauses](./testing-guard-clauses.md) — testing validation extensions
- [Testing LINQ Queries](./testing-linq-queries.md) — testing LINQ-style extensions
- [Testing Async Code](./testing-async-code.md) — testing async extensions
- [Testing String Assertions](./testing-string-assertions.md) — testing string extensions
