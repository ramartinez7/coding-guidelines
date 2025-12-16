# Testing Guard Clauses

> Verify input validation, precondition checks, and defensive programming using MSTest and FluentAssertions.

## Problem

Guard clauses protect methods from invalid inputs and states. Tests must verify that guards properly reject invalid data, throw appropriate exceptions, and allow valid inputs to proceed.

## Pattern

Test guard clauses for null checks, range validation, state verification, and combination conditions using FluentAssertions exception assertions.

## Example

### ❌ Before - No Guard Testing

```csharp
[TestMethod]
public void ProcessOrder_Works()
{
    var processor = new OrderProcessor();
    var result = processor.Process(order);
    Assert.IsTrue(result.IsSuccess);
    // ❌ Doesn't test guard clauses
}
```

### ✅ After - Comprehensive Guard Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

public class OrderProcessor
{
    public Result Process(Order order)
    {
        ArgumentNullException.ThrowIfNull(order, nameof(order));

        if (order.Total.Amount <= 0)
            return Result.Failure("Order total must be positive");

        if (order.Lines.Count == 0)
            return Result.Failure("Order must have at least one line");

        if (order.Status != OrderStatus.Pending)
            return Result.Failure("Only pending orders can be processed");

        // Processing logic...
        return Result.Success();
    }
}

[TestClass]
public class OrderProcessorGuardTests
{
    [TestMethod]
    public void Process_WithNullOrder_ThrowsArgumentNullException()
    {
        // Arrange
        var processor = new OrderProcessor();

        // Act
        Action act = () => processor.Process(null!);

        // Assert
        act.Should().Throw<ArgumentNullException>()
            .WithParameterName("order");
    }

    [TestMethod]
    public void Process_WithZeroTotal_ReturnsFailure()
    {
        // Arrange
        var processor = new OrderProcessor();
        var order = CreateOrder(total: Money.USD(0m));

        // Act
        var result = processor.Process(order);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("positive");
    }

    [TestMethod]
    public void Process_WithNoLines_ReturnsFailure()
    {
        // Arrange
        var processor = new OrderProcessor();
        var order = CreateOrderWithoutLines();

        // Act
        var result = processor.Process(order);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("at least one line");
    }

    [TestMethod]
    public void Process_WithNonPendingStatus_ReturnsFailure()
    {
        // Arrange
        var processor = new OrderProcessor();
        var order = CreateOrder(status: OrderStatus.Completed);

        // Act
        var result = processor.Process(order);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("pending");
    }

    [TestMethod]
    public void Process_WithValidOrder_Succeeds()
    {
        // Arrange
        var processor = new OrderProcessor();
        var order = CreateValidOrder();

        // Act
        var result = processor.Process(order);

        // Assert
        result.IsSuccess.Should().BeTrue();
    }
}
```

## Testing Null Guards

```csharp
public class CustomerService
{
    public Result SendEmail(Customer customer, string subject, string body)
    {
        ArgumentNullException.ThrowIfNull(customer, nameof(customer));
        ArgumentException.ThrowIfNullOrWhiteSpace(subject, nameof(subject));
        ArgumentException.ThrowIfNullOrWhiteSpace(body, nameof(body));

        // Send email logic...
        return Result.Success();
    }
}

[TestClass]
public class NullGuardTests
{
    [TestMethod]
    public void SendEmail_WithNullCustomer_ThrowsArgumentNullException()
    {
        // Arrange
        var service = new CustomerService();

        // Act
        Action act = () => service.SendEmail(null!, "Subject", "Body");

        // Assert
        act.Should().Throw<ArgumentNullException>()
            .WithParameterName("customer");
    }

    [TestMethod]
    public void SendEmail_WithNullSubject_ThrowsArgumentException()
    {
        // Arrange
        var service = new CustomerService();
        var customer = CreateCustomer();

        // Act
        Action act = () => service.SendEmail(customer, null!, "Body");

        // Assert
        act.Should().Throw<ArgumentException>()
            .WithParameterName("subject");
    }

    [TestMethod]
    public void SendEmail_WithEmptyBody_ThrowsArgumentException()
    {
        // Arrange
        var service = new CustomerService();
        var customer = CreateCustomer();

        // Act
        Action act = () => service.SendEmail(customer, "Subject", "");

        // Assert
        act.Should().Throw<ArgumentException>()
            .WithParameterName("body");
    }

    [TestMethod]
    public void SendEmail_WithWhitespaceBody_ThrowsArgumentException()
    {
        // Arrange
        var service = new CustomerService();
        var customer = CreateCustomer();

        // Act
        Action act = () => service.SendEmail(customer, "Subject", "   ");

        // Assert
        act.Should().Throw<ArgumentException>()
            .WithParameterName("body");
    }
}
```

## Testing Range Guards

```csharp
public class PaginationService
{
    private const int MaxPageSize = 100;

    public Result<Page<T>> GetPage<T>(
        IEnumerable<T> items,
        int pageNumber,
        int pageSize)
    {
        if (pageNumber < 1)
            return Result.Failure<Page<T>>("Page number must be at least 1");

        if (pageSize < 1)
            return Result.Failure<Page<T>>("Page size must be at least 1");

        if (pageSize > MaxPageSize)
            return Result.Failure<Page<T>>($"Page size cannot exceed {MaxPageSize}");

        // Pagination logic...
        return Result.Success(new Page<T>(items, pageNumber, pageSize));
    }
}

[TestClass]
public class RangeGuardTests
{
    [TestMethod]
    public void GetPage_WithZeroPageNumber_ReturnsFailure()
    {
        // Arrange
        var service = new PaginationService();
        var items = Enumerable.Range(1, 100);

        // Act
        var result = service.GetPage(items, pageNumber: 0, pageSize: 10);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("at least 1");
    }

    [TestMethod]
    public void GetPage_WithNegativePageSize_ReturnsFailure()
    {
        // Arrange
        var service = new PaginationService();
        var items = Enumerable.Range(1, 100);

        // Act
        var result = service.GetPage(items, pageNumber: 1, pageSize: -1);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("at least 1");
    }

    [TestMethod]
    public void GetPage_WithExcessivePageSize_ReturnsFailure()
    {
        // Arrange
        var service = new PaginationService();
        var items = Enumerable.Range(1, 100);

        // Act
        var result = service.GetPage(items, pageNumber: 1, pageSize: 101);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("cannot exceed 100");
    }

    [TestMethod]
    public void GetPage_WithValidParameters_Succeeds()
    {
        // Arrange
        var service = new PaginationService();
        var items = Enumerable.Range(1, 100);

        // Act
        var result = service.GetPage(items, pageNumber: 1, pageSize: 10);

        // Assert
        result.IsSuccess.Should().BeTrue();
    }
}
```

## Testing State Guards

```csharp
public class Order
{
    public OrderId Id { get; }
    public OrderStatus Status { get; private set; }

    public Result Ship()
    {
        if (Status != OrderStatus.Paid)
            return Result.Failure("Only paid orders can be shipped");

        Status = OrderStatus.Shipped;
        return Result.Success();
    }

    public Result Cancel()
    {
        if (Status == OrderStatus.Shipped || Status == OrderStatus.Delivered)
            return Result.Failure("Cannot cancel shipped or delivered orders");

        Status = OrderStatus.Cancelled;
        return Result.Success();
    }
}

[TestClass]
public class StateGuardTests
{
    [TestMethod]
    public void Ship_WhenNotPaid_ReturnsFailure()
    {
        // Arrange
        var order = CreateOrder(status: OrderStatus.Pending);

        // Act
        var result = order.Ship();

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("paid");
        order.Status.Should().Be(OrderStatus.Pending);
    }

    [TestMethod]
    public void Ship_WhenPaid_Succeeds()
    {
        // Arrange
        var order = CreateOrder(status: OrderStatus.Paid);

        // Act
        var result = order.Ship();

        // Assert
        result.IsSuccess.Should().BeTrue();
        order.Status.Should().Be(OrderStatus.Shipped);
    }

    [TestMethod]
    public void Cancel_WhenShipped_ReturnsFailure()
    {
        // Arrange
        var order = CreateOrder(status: OrderStatus.Shipped);

        // Act
        var result = order.Cancel();

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("Cannot cancel");
        order.Status.Should().Be(OrderStatus.Shipped);
    }

    [TestMethod]
    public void Cancel_WhenPending_Succeeds()
    {
        // Arrange
        var order = CreateOrder(status: OrderStatus.Pending);

        // Act
        var result = order.Cancel();

        // Assert
        result.IsSuccess.Should().BeTrue();
        order.Status.Should().Be(OrderStatus.Cancelled);
    }
}
```

## Testing Collection Guards

```csharp
public class BatchProcessor
{
    private const int MaxBatchSize = 1000;

    public Result ProcessBatch(IEnumerable<Order> orders)
    {
        ArgumentNullException.ThrowIfNull(orders, nameof(orders));

        var orderList = orders.ToList();

        if (orderList.Count == 0)
            return Result.Failure("Batch cannot be empty");

        if (orderList.Count > MaxBatchSize)
            return Result.Failure($"Batch size cannot exceed {MaxBatchSize}");

        if (orderList.Any(o => o == null))
            return Result.Failure("Batch cannot contain null orders");

        // Processing logic...
        return Result.Success();
    }
}

[TestClass]
public class CollectionGuardTests
{
    [TestMethod]
    public void ProcessBatch_WithNullCollection_ThrowsArgumentNullException()
    {
        // Arrange
        var processor = new BatchProcessor();

        // Act
        Action act = () => processor.ProcessBatch(null!);

        // Assert
        act.Should().Throw<ArgumentNullException>()
            .WithParameterName("orders");
    }

    [TestMethod]
    public void ProcessBatch_WithEmptyCollection_ReturnsFailure()
    {
        // Arrange
        var processor = new BatchProcessor();
        var orders = Enumerable.Empty<Order>();

        // Act
        var result = processor.ProcessBatch(orders);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("empty");
    }

    [TestMethod]
    public void ProcessBatch_WithExcessiveSize_ReturnsFailure()
    {
        // Arrange
        var processor = new BatchProcessor();
        var orders = Enumerable.Range(1, 1001).Select(_ => CreateOrder());

        // Act
        var result = processor.ProcessBatch(orders);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("cannot exceed");
    }

    [TestMethod]
    public void ProcessBatch_WithNullElement_ReturnsFailure()
    {
        // Arrange
        var processor = new BatchProcessor();
        var orders = new[] { CreateOrder(), null!, CreateOrder() };

        // Act
        var result = processor.ProcessBatch(orders);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("null");
    }
}
```

## Testing Combined Guards

```csharp
public class TransferService
{
    public Result Transfer(BankAccount from, BankAccount to, Money amount)
    {
        ArgumentNullException.ThrowIfNull(from, nameof(from));
        ArgumentNullException.ThrowIfNull(to, nameof(to));
        ArgumentNullException.ThrowIfNull(amount, nameof(amount));

        if (amount.Amount <= 0)
            return Result.Failure("Transfer amount must be positive");

        if (from.Id == to.Id)
            return Result.Failure("Cannot transfer to same account");

        if (from.Balance.Amount < amount.Amount)
            return Result.Failure("Insufficient funds");

        if (from.Currency != amount.Currency || to.Currency != amount.Currency)
            return Result.Failure("Currency mismatch");

        // Transfer logic...
        return Result.Success();
    }
}

[TestClass]
public class CombinedGuardTests
{
    [TestMethod]
    public void Transfer_WithAllValidConditions_Succeeds()
    {
        // Arrange
        var service = new TransferService();
        var from = CreateAccount(balance: Money.USD(1000m));
        var to = CreateAccount(balance: Money.USD(0m));
        var amount = Money.USD(100m);

        // Act
        var result = service.Transfer(from, to, amount);

        // Assert
        result.IsSuccess.Should().BeTrue();
    }

    [TestMethod]
    public void Transfer_ToSameAccount_ReturnsFailure()
    {
        // Arrange
        var service = new TransferService();
        var account = CreateAccount();
        var amount = Money.USD(100m);

        // Act
        var result = service.Transfer(account, account, amount);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("same account");
    }

    [TestMethod]
    public void Transfer_WithInsufficientFunds_ReturnsFailure()
    {
        // Arrange
        var service = new TransferService();
        var from = CreateAccount(balance: Money.USD(50m));
        var to = CreateAccount();
        var amount = Money.USD(100m);

        // Act
        var result = service.Transfer(from, to, amount);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("Insufficient");
    }

    [TestMethod]
    public void Transfer_WithCurrencyMismatch_ReturnsFailure()
    {
        // Arrange
        var service = new TransferService();
        var from = CreateAccount(balance: Money.USD(1000m));
        var to = CreateAccount(balance: Money.EUR(0m));
        var amount = Money.USD(100m);

        // Act
        var result = service.Transfer(from, to, amount);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("Currency");
    }
}
```

## Why It's Important

1. **Defensive Programming**: Prevent invalid states
2. **Clear Errors**: Provide specific error messages
3. **Early Validation**: Fail fast with bad inputs
4. **Documentation**: Guards document preconditions
5. **Robustness**: Handle edge cases explicitly

## Guidelines

**Testing Guards**
- Test each guard condition independently
- Test combinations of conditions
- Verify exception types and messages
- Test valid inputs pass all guards
- Check parameter names in exceptions

**Common Guard Patterns**
- Null checks: `ArgumentNullException.ThrowIfNull`
- String validation: `ArgumentException.ThrowIfNullOrWhiteSpace`
- Range validation: Compare against min/max
- State validation: Check current state
- Collection validation: Size and null elements

**Assertions**
- `.Should().Throw<ArgumentNullException>()`
- `.WithParameterName("paramName")`
- `.IsFailure.Should().BeTrue()`
- `.Error.Should().Contain("expected text")`

## Common Pitfalls

❌ **Not testing all guards**
```csharp
// Only tests one condition
act.Should().Throw<ArgumentNullException>();
```

✅ **Test each guard**
```csharp
// Test null check
service.Method(null).Should().Throw<ArgumentNullException>();
// Test empty check  
service.Method("").Should().Throw<ArgumentException>();
// Test whitespace check
service.Method("   ").Should().Throw<ArgumentException>();
```

❌ **Not verifying parameter names**
```csharp
// Incomplete
act.Should().Throw<ArgumentNullException>();
```

✅ **Verify parameter name**
```csharp
// Complete
act.Should().Throw<ArgumentNullException>()
    .WithParameterName("customer");
```

## See Also

- [Testing Domain Invariants](./testing-domain-invariants.md) — business rules
- [Testing Exceptions](./testing-exceptions.md) — exception testing
- [Smart Constructors](../patterns/smart-constructors.md) — validated construction
- [Honest Functions](../patterns/honest-functions.md) — explicit error handling
