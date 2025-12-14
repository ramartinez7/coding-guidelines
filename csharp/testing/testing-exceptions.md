# Testing Exceptions

> Verify error conditions—test both that exceptions are thrown and that they contain the right information.

## Problem

Tests that only verify exceptions are thrown miss important details like error messages, error codes, and exception data. Tests that don't verify exceptions at all miss critical error paths.

## Pattern

Test both the happy path and error paths. Verify exception type, message, and any custom properties.

## Example

### ❌ Before - Incomplete Exception Testing

```csharp
[Fact]
public void CreateOrder_WithNegativeAmount_Throws()
{
    Assert.Throws<Exception>(() => // ❌ Too generic
        Order.Create(CustomerId.New(), Money.USD(-100m))
    );
}
```

### ✅ After - Complete Exception Testing

```csharp
[Fact]
public void CreateOrder_WithNegativeAmount_ThrowsArgumentException()
{
    // Arrange
    var customerId = CustomerId.New();
    var negativeAmount = Money.USD(-100m);

    // Act & Assert
    var exception = Assert.Throws<ArgumentException>(() =>
        Order.Create(customerId, negativeAmount)
    );

    Assert.Equal("amount", exception.ParamName);
    Assert.Contains("negative", exception.Message, StringComparison.OrdinalIgnoreCase);
}
```

## Testing Exception Types

### Specific Exception Type

```csharp
[Fact]
public void Divide_ByZero_ThrowsDivideByZeroException()
{
    // Arrange
    var calculator = new Calculator();

    // Act & Assert
    Assert.Throws<DivideByZeroException>(() =>
        calculator.Divide(10, 0)
    );
}
```

### Custom Domain Exceptions

```csharp
[Fact]
public void ProcessPayment_WithInsufficientFunds_ThrowsInsufficientFundsException()
{
    // Arrange
    var account = Account.Create(AccountId.New(), Money.USD(50m));
    var service = new PaymentService();

    // Act & Assert
    var exception = Assert.Throws<InsufficientFundsException>(() =>
        service.ProcessPayment(account, Money.USD(100m))
    );

    Assert.Equal(account.Id, exception.AccountId);
    Assert.Equal(Money.USD(50m), exception.Available);
    Assert.Equal(Money.USD(100m), exception.Required);
}
```

## Testing Exception Messages

```csharp
[Fact]
public void CreateEmailAddress_WithInvalidFormat_ThrowsWithDescriptiveMessage()
{
    // Arrange
    var invalidEmail = "not-an-email";

    // Act & Assert
    var exception = Assert.Throws<ArgumentException>(() =>
        EmailAddress.FromString(invalidEmail)
    );

    Assert.Contains("email", exception.Message, StringComparison.OrdinalIgnoreCase);
    Assert.Contains("format", exception.Message, StringComparison.OrdinalIgnoreCase);
    Assert.Contains(invalidEmail, exception.Message);
}
```

## Testing ArgumentException with ParamName

```csharp
[Fact]
public void CreateOrder_WithNullCustomerId_ThrowsArgumentNullException()
{
    // Act & Assert
    var exception = Assert.Throws<ArgumentNullException>(() =>
        Order.Create(customerId: null!, Money.USD(100m))
    );

    Assert.Equal("customerId", exception.ParamName);
}

[Theory]
[InlineData(-1, "age")]
[InlineData(-100, "age")]
[InlineData(200, "age")]
public void CreateAge_WithInvalidValue_ThrowsArgumentOutOfRangeException(
    int invalidAge, 
    string expectedParamName)
{
    // Act & Assert
    var exception = Assert.Throws<ArgumentOutOfRangeException>(() =>
        Age.FromInt(invalidAge)
    );

    Assert.Equal(expectedParamName, exception.ParamName);
    Assert.Equal(invalidAge, exception.ActualValue);
}
```

## Testing Async Exceptions

```csharp
[Fact]
public async Task ProcessOrderAsync_WithInvalidOrder_ThrowsInvalidOperationException()
{
    // Arrange
    var service = new OrderService(repository);
    var invalidOrder = OrderBuilder.Default()
        .WithStatus(OrderStatus.Cancelled)
        .Build();

    // Act & Assert
    await Assert.ThrowsAsync<InvalidOperationException>(async () =>
        await service.ProcessAsync(invalidOrder.Id)
    );
}

[Fact]
public async Task SendEmailAsync_WithInvalidAddress_ThrowsEmailException()
{
    // Arrange
    var service = new EmailService();
    var invalidEmail = "not-an-email";

    // Act & Assert
    var exception = await Assert.ThrowsAsync<EmailException>(async () =>
        await service.SendAsync(invalidEmail, "Subject", "Body")
    );

    Assert.Equal(EmailErrorCode.InvalidAddress, exception.ErrorCode);
    Assert.Contains(invalidEmail, exception.Message);
}
```

## Testing Exception Inner Exceptions

```csharp
[Fact]
public void ProcessOrder_WhenRepositoryFails_WrapsException()
{
    // Arrange
    var failingRepository = new FailingOrderRepository();
    var service = new OrderService(failingRepository);

    // Act & Assert
    var exception = Assert.Throws<OrderProcessingException>(() =>
        service.Process(OrderId.New())
    );

    Assert.NotNull(exception.InnerException);
    Assert.IsType<DatabaseException>(exception.InnerException);
    Assert.Contains("database", exception.Message, StringComparison.OrdinalIgnoreCase);
}
```

## Testing Multiple Exception Conditions

```csharp
public class EmailAddressValidationTests
{
    [Theory]
    [InlineData(null, "Value cannot be null")]
    [InlineData("", "Value cannot be empty")]
    [InlineData("   ", "Value cannot be whitespace")]
    [InlineData("invalid", "must contain @")]
    [InlineData("@example.com", "local part cannot be empty")]
    [InlineData("user@", "domain cannot be empty")]
    public void FromString_WithInvalidInput_ThrowsWithCorrectMessage(
        string invalidInput,
        string expectedMessagePart)
    {
        // Act & Assert
        var exception = Assert.Throws<ArgumentException>(() =>
            EmailAddress.FromString(invalidInput)
        );

        Assert.Contains(expectedMessagePart, exception.Message, StringComparison.OrdinalIgnoreCase);
    }
}
```

## Testing Exception vs. Result Pattern

When using the Result pattern, test both throwing and non-throwing versions:

```csharp
// Static factory that throws
[Fact]
public void FromString_WithInvalidEmail_ThrowsException()
{
    var exception = Assert.Throws<ArgumentException>(() =>
        EmailAddress.FromString("invalid")
    );
    Assert.Contains("email", exception.Message, StringComparison.OrdinalIgnoreCase);
}

// TryParse that returns bool
[Fact]
public void TryFromString_WithInvalidEmail_ReturnsFalse()
{
    var success = EmailAddress.TryFromString("invalid", out var email);
    
    Assert.False(success);
    Assert.Equal(default, email);
}

// Create that returns Result
[Fact]
public void Create_WithInvalidEmail_ReturnsFailure()
{
    var result = EmailAddress.Create("invalid");
    
    Assert.True(result.IsFailure);
    Assert.Contains("email", result.Error, StringComparison.OrdinalIgnoreCase);
}
```

## Testing Guard Clauses

```csharp
[Fact]
public void Constructor_WithNullRepository_ThrowsArgumentNullException()
{
    // Act & Assert
    var exception = Assert.Throws<ArgumentNullException>(() =>
        new OrderService(repository: null!)
    );

    Assert.Equal("repository", exception.ParamName);
}

[Theory]
[InlineData(null, "customerId")]
[InlineData("amount")]
public void CreateOrder_WithNullParameters_ThrowsArgumentNullException(string expectedParamName)
{
    // This would need separate tests for each parameter
}
```

## Testing Aggregate Exceptions

```csharp
[Fact]
public async Task ProcessMultipleOrders_WhenSomeFail_ThrowsAggregateException()
{
    // Arrange
    var service = new OrderService(repository);
    var orders = new[] 
    { 
        ValidOrderId(), 
        InvalidOrderId(), 
        AnotherInvalidOrderId() 
    };

    // Act & Assert
    var exception = await Assert.ThrowsAsync<AggregateException>(async () =>
        await service.ProcessMultipleAsync(orders)
    );

    Assert.Equal(2, exception.InnerExceptions.Count);
    Assert.All(exception.InnerExceptions, 
        ex => Assert.IsType<InvalidOperationException>(ex));
}
```

## Testing Exception Data

```csharp
[Fact]
public void ProcessOrder_WhenValidationFails_IncludesValidationErrors()
{
    // Arrange
    var service = new OrderService(repository);
    var invalidOrder = OrderBuilder.Default()
        .WithEmptyCart()
        .Build();

    // Act & Assert
    var exception = Assert.Throws<ValidationException>(() =>
        service.Process(invalidOrder)
    );

    Assert.NotEmpty(exception.Errors);
    Assert.Contains(exception.Errors, 
        e => e.PropertyName == "Items" && e.ErrorMessage.Contains("empty"));
}
```

## Don't Test Implementation Details

```csharp
// ❌ Bad - Testing exact exception message
[Fact]
public void Test()
{
    var exception = Assert.Throws<Exception>(() => DoSomething());
    Assert.Equal("The exact error message", exception.Message); // Brittle
}

// ✅ Good - Testing that message contains key information
[Fact]
public void Test()
{
    var exception = Assert.Throws<DomainException>(() => DoSomething());
    Assert.Contains("customer", exception.Message, StringComparison.OrdinalIgnoreCase);
    Assert.Equal(ErrorCode.InvalidCustomer, exception.ErrorCode); // Test structured data
}
```

## Testing That Exceptions Are Not Thrown

```csharp
[Fact]
public void CreateOrder_WithValidInputs_DoesNotThrow()
{
    // Arrange
    var customerId = CustomerId.New();
    var amount = Money.USD(100m);

    // Act
    var exception = Record.Exception(() =>
        Order.Create(customerId, amount)
    );

    // Assert
    Assert.Null(exception);
}
```

## Guidelines

1. **Test specific exception types**, not just `Exception`
2. **Verify exception messages** contain meaningful information
3. **Check ParamName** for argument exceptions
4. **Test custom exception properties** (error codes, context data)
5. **Use Result<T> pattern** for expected errors instead of exceptions
6. **Test both throwing and non-throwing APIs** when both exist

## Benefits

1. **Verification**: Ensures errors are reported correctly
2. **Documentation**: Tests show what errors are possible
3. **Debugging**: Specific assertions help locate failures
4. **Maintenance**: Catch breaking changes to error handling
5. **User experience**: Better error messages improve usability

## See Also

- [Error Handling](../patterns/error-handling.md) — production error handling patterns
- [Result Monad](../patterns/result-monad.md) — alternatives to exceptions
- [Honest Functions](../patterns/honest-functions.md) — making errors explicit
- [Testing Async Code](./testing-async-code.md) — async exception testing
