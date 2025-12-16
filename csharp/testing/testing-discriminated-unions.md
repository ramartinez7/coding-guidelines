# Testing Discriminated Unions

> Exhaustively test all variants—ensure pattern matching handles every case correctly.

## Problem

Discriminated unions (sum types) have multiple variants. Tests must verify each variant's behavior and ensure pattern matching is exhaustive.

## Pattern

Test each variant independently, verify pattern matching handles all cases, and ensure type-safe operations work correctly for each variant.

## Example

### ❌ Before - Incomplete Variant Testing

```csharp
[Fact]
public void ProcessPaymentResult_Works()
{
    var result = PaymentResult.Success(TransactionId.New());
    Assert.True(result.IsSuccess);
}
// ❌ Missing: testing other variants, pattern matching
```

### ✅ After - Exhaustive Variant Testing

```csharp
[Fact]
public void Success_Variant_HasTransactionId()
{
    var transactionId = TransactionId.New();
    
    var result = PaymentResult.Success(transactionId);
    
    Assert.True(result.IsSuccess);
    var successResult = Assert.IsType<PaymentResult.Succeeded>(result);
    Assert.Equal(transactionId, successResult.TransactionId);
}

[Fact]
public void Failure_Variant_HasErrorCode()
{
    var errorCode = PaymentErrorCode.InsufficientFunds;
    var message = "Insufficient funds";
    
    var result = PaymentResult.Failure(errorCode, message);
    
    Assert.True(result.IsFailure);
    var failureResult = Assert.IsType<PaymentResult.Failed>(result);
    Assert.Equal(errorCode, failureResult.ErrorCode);
    Assert.Equal(message, failureResult.Message);
}

[Fact]
public void Pending_Variant_HasReferenceNumber()
{
    var referenceNumber = "REF123";
    
    var result = PaymentResult.Pending(referenceNumber);
    
    var pendingResult = Assert.IsType<PaymentResult.PendingApproval>(result);
    Assert.Equal(referenceNumber, pendingResult.ReferenceNumber);
}
```

## Defining Discriminated Unions in C#

### Using Abstract Record Base Class

```csharp
public abstract record PaymentResult
{
    public sealed record Succeeded(TransactionId TransactionId, DateTime ProcessedAt) 
        : PaymentResult;
    
    public sealed record Failed(PaymentErrorCode ErrorCode, string Message) 
        : PaymentResult;
    
    public sealed record PendingApproval(string ReferenceNumber, DateTime RequestedAt) 
        : PaymentResult;

    // Prevent inheritance outside this type
    private PaymentResult() { }
}
```

### Testing Each Variant

```csharp
public class PaymentResultTests
{
    [Fact]
    public void Succeeded_Variant_ContainsTransactionDetails()
    {
        // Arrange
        var transactionId = TransactionId.New();
        var processedAt = DateTime.UtcNow;

        // Act
        var result = new PaymentResult.Succeeded(transactionId, processedAt);

        // Assert
        Assert.Equal(transactionId, result.TransactionId);
        Assert.Equal(processedAt, result.ProcessedAt);
    }

    [Fact]
    public void Failed_Variant_ContainsErrorInformation()
    {
        // Arrange
        var errorCode = PaymentErrorCode.InvalidCard;
        var message = "Card number is invalid";

        // Act
        var result = new PaymentResult.Failed(errorCode, message);

        // Assert
        Assert.Equal(errorCode, result.ErrorCode);
        Assert.Equal(message, result.Message);
    }

    [Fact]
    public void PendingApproval_Variant_ContainsReferenceData()
    {
        // Arrange
        var referenceNumber = "REF-123456";
        var requestedAt = DateTime.UtcNow;

        // Act
        var result = new PaymentResult.PendingApproval(referenceNumber, requestedAt);

        // Assert
        Assert.Equal(referenceNumber, result.ReferenceNumber);
        Assert.Equal(requestedAt, result.RequestedAt);
    }
}
```

## Testing Pattern Matching

### Switch Expressions

```csharp
[Fact]
public void PatternMatch_HandlesAllVariants()
{
    // Arrange
    var results = new PaymentResult[]
    {
        new PaymentResult.Succeeded(TransactionId.New(), DateTime.UtcNow),
        new PaymentResult.Failed(PaymentErrorCode.InsufficientFunds, "Not enough funds"),
        new PaymentResult.PendingApproval("REF123", DateTime.UtcNow),
    };

    // Act & Assert - Verify all variants handled
    foreach (var result in results)
    {
        var message = result switch
        {
            PaymentResult.Succeeded s => $"Success: {s.TransactionId}",
            PaymentResult.Failed f => $"Failed: {f.ErrorCode}",
            PaymentResult.PendingApproval p => $"Pending: {p.ReferenceNumber}",
            _ => throw new InvalidOperationException("Unhandled variant")
        };

        Assert.NotNull(message);
        Assert.NotEmpty(message);
    }
}

[Fact]
public void GetDisplayMessage_ReturnsCorrectMessageForEachVariant()
{
    // Test that business logic handles all variants
    var succeeded = new PaymentResult.Succeeded(TransactionId.New(), DateTime.UtcNow);
    var failed = new PaymentResult.Failed(PaymentErrorCode.Declined, "Card declined");
    var pending = new PaymentResult.PendingApproval("REF123", DateTime.UtcNow);

    Assert.Contains("successful", GetDisplayMessage(succeeded), StringComparison.OrdinalIgnoreCase);
    Assert.Contains("declined", GetDisplayMessage(failed), StringComparison.OrdinalIgnoreCase);
    Assert.Contains("pending", GetDisplayMessage(pending), StringComparison.OrdinalIgnoreCase);
}

private static string GetDisplayMessage(PaymentResult result) => result switch
{
    PaymentResult.Succeeded => "Payment successful",
    PaymentResult.Failed f => $"Payment failed: {f.Message}",
    PaymentResult.PendingApproval => "Payment pending approval",
    _ => throw new InvalidOperationException("Unhandled payment result")
};
```

## Testing Variant Equality

```csharp
[Fact]
public void Variants_WithSameValues_AreEqual()
{
    var transactionId = TransactionId.Parse("00000000-0000-0000-0000-000000000001");
    var processedAt = new DateTime(2024, 1, 1, 12, 0, 0, DateTimeKind.Utc);

    var result1 = new PaymentResult.Succeeded(transactionId, processedAt);
    var result2 = new PaymentResult.Succeeded(transactionId, processedAt);

    Assert.Equal(result1, result2);
    Assert.True(result1 == result2);
}

[Fact]
public void Variants_WithDifferentValues_AreNotEqual()
{
    var result1 = new PaymentResult.Succeeded(TransactionId.New(), DateTime.UtcNow);
    var result2 = new PaymentResult.Succeeded(TransactionId.New(), DateTime.UtcNow);

    Assert.NotEqual(result1, result2);
}

[Fact]
public void DifferentVariants_AreNeverEqual()
{
    var succeeded = new PaymentResult.Succeeded(TransactionId.New(), DateTime.UtcNow);
    var failed = new PaymentResult.Failed(PaymentErrorCode.Declined, "Declined");

    Assert.NotEqual(succeeded, failed);
}
```

## Testing Complex Discriminated Unions

```csharp
public abstract record OrderEvent
{
    public sealed record Created(OrderId OrderId, CustomerId CustomerId, DateTime CreatedAt) 
        : OrderEvent;
    
    public sealed record ItemAdded(OrderId OrderId, ProductId ProductId, int Quantity, DateTime AddedAt) 
        : OrderEvent;
    
    public sealed record ItemRemoved(OrderId OrderId, ProductId ProductId, DateTime RemovedAt) 
        : OrderEvent;
    
    public sealed record Confirmed(OrderId OrderId, Money Total, DateTime ConfirmedAt) 
        : OrderEvent;
    
    public sealed record Cancelled(OrderId OrderId, string Reason, DateTime CancelledAt) 
        : OrderEvent;

    private OrderEvent() { }
}

public class OrderEventTests
{
    [Fact]
    public void Created_Event_ContainsRequiredData()
    {
        var orderId = OrderId.New();
        var customerId = CustomerId.New();
        var createdAt = DateTime.UtcNow;

        var evt = new OrderEvent.Created(orderId, customerId, createdAt);

        Assert.Equal(orderId, evt.OrderId);
        Assert.Equal(customerId, evt.CustomerId);
        Assert.Equal(createdAt, evt.CreatedAt);
    }

    [Theory]
    [MemberData(nameof(AllEventVariants))]
    public void AllVariants_CanBeSerialized(OrderEvent evt)
    {
        var json = JsonSerializer.Serialize(evt, new JsonSerializerOptions
        {
            WriteIndented = true
        });

        Assert.NotNull(json);
        Assert.NotEmpty(json);
    }

    public static IEnumerable<object[]> AllEventVariants()
    {
        yield return new object[] { new OrderEvent.Created(OrderId.New(), CustomerId.New(), DateTime.UtcNow) };
        yield return new object[] { new OrderEvent.ItemAdded(OrderId.New(), ProductId.New(), 1, DateTime.UtcNow) };
        yield return new object[] { new OrderEvent.ItemRemoved(OrderId.New(), ProductId.New(), DateTime.UtcNow) };
        yield return new object[] { new OrderEvent.Confirmed(OrderId.New(), Money.USD(100m), DateTime.UtcNow) };
        yield return new object[] { new OrderEvent.Cancelled(OrderId.New(), "Customer request", DateTime.UtcNow) };
    }
}
```

## Testing Result<T, E> Pattern

```csharp
public abstract record Result<TValue, TError>
{
    public sealed record Success(TValue Value) : Result<TValue, TError>;
    public sealed record Failure(TError Error) : Result<TValue, TError>;

    private Result() { }
}

public class ResultTests
{
    [Fact]
    public void Success_Variant_ContainsValue()
    {
        var value = 42;
        
        var result = new Result<int, string>.Success(value);
        
        Assert.Equal(value, result.Value);
    }

    [Fact]
    public void Failure_Variant_ContainsError()
    {
        var error = "Something went wrong";
        
        var result = new Result<int, string>.Failure(error);
        
        Assert.Equal(error, result.Error);
    }

    [Fact]
    public void Map_OnSuccess_TransformsValue()
    {
        var result = new Result<int, string>.Success(5);

        var mapped = result switch
        {
            Result<int, string>.Success s => new Result<string, string>.Success(s.Value.ToString()),
            Result<int, string>.Failure f => new Result<string, string>.Failure(f.Error),
            _ => throw new InvalidOperationException()
        };

        var successMapped = Assert.IsType<Result<string, string>.Success>(mapped);
        Assert.Equal("5", successMapped.Value);
    }

    [Fact]
    public void Map_OnFailure_PreservesError()
    {
        var result = new Result<int, string>.Failure("Error");

        var mapped = result switch
        {
            Result<int, string>.Success s => new Result<string, string>.Success(s.Value.ToString()),
            Result<int, string>.Failure f => new Result<string, string>.Failure(f.Error),
            _ => throw new InvalidOperationException()
        };

        var failureMapped = Assert.IsType<Result<string, string>.Failure>(mapped);
        Assert.Equal("Error", failureMapped.Error);
    }
}
```

## Testing Type Guards

```csharp
[Fact]
public void IsSuccess_OnSuccessVariant_ReturnsTrue()
{
    var result = new Result<int, string>.Success(42);
    
    bool isSuccess = result is Result<int, string>.Success;
    
    Assert.True(isSuccess);
}

[Fact]
public void IsFailure_OnFailureVariant_ReturnsTrue()
{
    var result = new Result<int, string>.Failure("Error");
    
    bool isFailure = result is Result<int, string>.Failure;
    
    Assert.True(isFailure);
}
```

## Testing Nested Discriminated Unions

```csharp
public abstract record ValidationError
{
    public sealed record Required(string FieldName) : ValidationError;
    public sealed record InvalidFormat(string FieldName, string ExpectedFormat) : ValidationError;
    public sealed record OutOfRange(string FieldName, object Min, object Max, object Actual) : ValidationError;
    
    private ValidationError() { }
}

[Fact]
public void ValidationErrors_AllVariants_ProduceCorrectMessages()
{
    var errors = new ValidationError[]
    {
        new ValidationError.Required("Email"),
        new ValidationError.InvalidFormat("Phone", "XXX-XXX-XXXX"),
        new ValidationError.OutOfRange("Age", 0, 120, 150),
    };

    foreach (var error in errors)
    {
        var message = error switch
        {
            ValidationError.Required r => $"{r.FieldName} is required",
            ValidationError.InvalidFormat f => $"{f.FieldName} must match format: {f.ExpectedFormat}",
            ValidationError.OutOfRange r => $"{r.FieldName} must be between {r.Min} and {r.Max}, got {r.Actual}",
            _ => throw new InvalidOperationException("Unhandled validation error")
        };

        Assert.NotNull(message);
        Assert.NotEmpty(message);
    }
}
```

## Guidelines

1. **Test each variant independently**
2. **Verify pattern matching is exhaustive**
3. **Test variant equality**
4. **Use parameterized tests for all variants**
5. **Test serialization/deserialization**
6. **Verify type guards work correctly**

## Benefits

1. **Type safety**: Compiler enforces exhaustive matching
2. **Correctness**: All cases are tested
3. **Maintainability**: Adding variants forces test updates
4. **Documentation**: Tests show all possible cases
5. **Confidence**: Know all variants work correctly

## See Also

- [Testing State Machines](./testing-state-machines.md) — state as discriminated union
- [Discriminated Unions](../patterns/discriminated-unions.md) — design pattern
- [Exhaustive Pattern Matching](../patterns/exhaustive-pattern-matching.md) — compiler enforcement
- [Result Monad](../patterns/result-monad.md) — error handling with unions
