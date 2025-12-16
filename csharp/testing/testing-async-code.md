# Testing Async Code

> Properly test asynchronous operations—await async methods and verify timing-dependent behavior.

## Problem

Async tests that don't properly await can pass even when the code is broken. Race conditions and timing issues are hard to reproduce and test reliably.

## Pattern

Always use async test methods, properly await async operations, and use appropriate techniques for testing timing-dependent behavior.

## Example

### ❌ Before - Not Awaiting

```csharp
[Fact]
public void ProcessOrder_SavesOrder() // ❌ Not async
{
    var service = new OrderService(repository);
    
    service.ProcessOrderAsync(orderId); // ❌ Not awaited - test passes even if it fails!
    
    var saved = repository.Get(orderId);
    Assert.NotNull(saved);
}
```

### ✅ After - Proper Async Test

```csharp
[Fact]
public async Task ProcessOrder_SavesOrder() // ✅ Async test
{
    // Arrange
    var repository = new InMemoryOrderRepository();
    var service = new OrderService(repository);
    var orderId = OrderId.New();

    // Act
    await service.ProcessOrderAsync(orderId); // ✅ Awaited

    // Assert
    var saved = await repository.GetAsync(orderId);
    Assert.NotNull(saved);
}
```

## Testing Async Methods

### Basic Async Test

```csharp
[Fact]
public async Task SendEmailAsync_WithValidEmail_Succeeds()
{
    // Arrange
    var emailService = new EmailService();
    var email = EmailAddress.Create("test@example.com").Value;

    // Act
    var result = await emailService.SendAsync(email, "Subject", "Body");

    // Assert
    Assert.True(result.IsSuccess);
}
```

### Testing Result<T> with Async

```csharp
[Fact]
public async Task CreateOrderAsync_WithValidInput_ReturnsSuccess()
{
    // Arrange
    var service = new OrderService(repository);
    var customerId = CustomerId.New();
    var items = OrderItemsBuilder.Default().Build();

    // Act
    var result = await service.CreateOrderAsync(customerId, items);

    // Assert
    Assert.True(result.IsSuccess);
    Assert.NotNull(result.Value);
    Assert.Equal(customerId, result.Value.CustomerId);
}

[Fact]
public async Task CreateOrderAsync_WithEmptyItems_ReturnsFailure()
{
    // Arrange
    var service = new OrderService(repository);
    var customerId = CustomerId.New();
    var emptyItems = new List<OrderItem>();

    // Act
    var result = await service.CreateOrderAsync(customerId, emptyItems);

    // Assert
    Assert.True(result.IsFailure);
    Assert.Contains("at least one item", result.Error);
}
```

## Testing Exceptions

```csharp
[Fact]
public async Task ProcessPaymentAsync_WithInvalidCard_ThrowsException()
{
    // Arrange
    var service = new PaymentService();
    var invalidCard = CardNumber.Create("0000000000000000").Value;

    // Act & Assert
    await Assert.ThrowsAsync<PaymentException>(
        async () => await service.ProcessAsync(invalidCard, Money.USD(100m))
    );
}

[Fact]
public async Task ProcessPaymentAsync_WithInvalidCard_ThrowsCorrectException()
{
    // Arrange
    var service = new PaymentService();
    var invalidCard = CardNumber.Create("0000000000000000").Value;

    // Act
    var exception = await Assert.ThrowsAsync<PaymentException>(
        async () => await service.ProcessAsync(invalidCard, Money.USD(100m))
    );

    // Assert
    Assert.Equal(PaymentErrorCode.InvalidCard, exception.ErrorCode);
    Assert.Contains("invalid", exception.Message, StringComparison.OrdinalIgnoreCase);
}
```

## Testing Cancellation

```csharp
[Fact]
public async Task ProcessLongRunningOperation_WhenCancelled_ThrowsOperationCanceledException()
{
    // Arrange
    var service = new DataProcessingService();
    var cts = new CancellationTokenSource();
    
    // Act
    var task = service.ProcessAsync(largeDataSet, cts.Token);
    cts.Cancel(); // Cancel immediately

    // Assert
    await Assert.ThrowsAsync<OperationCanceledException>(async () => await task);
}

[Fact]
public async Task ProcessWithCancellation_WhenCancelled_CleansUpResources()
{
    // Arrange
    var service = new DataProcessingService();
    var cts = new CancellationTokenSource();

    // Act
    var task = service.ProcessAsync(largeDataSet, cts.Token);
    await Task.Delay(100); // Let it start
    cts.Cancel();

    try
    {
        await task;
    }
    catch (OperationCanceledException)
    {
        // Expected
    }

    // Assert
    Assert.True(service.ResourcesReleased);
}
```

## Testing Timeouts

```csharp
[Fact(Timeout = 5000)] // Fail if test takes longer than 5 seconds
public async Task FetchDataAsync_CompletesWithinTimeout()
{
    // Arrange
    var service = new DataService();

    // Act
    var result = await service.FetchDataAsync();

    // Assert
    Assert.NotNull(result);
}

[Fact]
public async Task FetchDataAsync_WithSlowResponse_TimesOut()
{
    // Arrange
    var service = new DataService(responseDelay: TimeSpan.FromSeconds(10));
    var cts = new CancellationTokenSource(TimeSpan.FromSeconds(1));

    // Act & Assert
    await Assert.ThrowsAsync<OperationCanceledException>(
        async () => await service.FetchDataAsync(cts.Token)
    );
}
```

## Testing Concurrent Operations

```csharp
[Fact]
public async Task ProcessMultipleOrdersConcurrently_AllSucceed()
{
    // Arrange
    var service = new OrderService(repository);
    var orders = Enumerable.Range(1, 10)
        .Select(_ => OrderBuilder.Default().Build())
        .ToList();

    // Act
    var tasks = orders.Select(o => service.ProcessAsync(o.Id));
    var results = await Task.WhenAll(tasks);

    // Assert
    Assert.All(results, r => Assert.True(r.IsSuccess));
}

[Fact]
public async Task ConcurrentAccess_MaintainsDataIntegrity()
{
    // Arrange
    var account = Account.Create(AccountId.New(), Money.USD(1000m));
    var service = new AccountService(repository);

    // Act - 10 concurrent withdrawals of $100
    var tasks = Enumerable.Range(1, 10)
        .Select(_ => service.WithdrawAsync(account.Id, Money.USD(100m)));
    
    var results = await Task.WhenAll(tasks);

    // Assert
    var finalAccount = await repository.GetAsync(account.Id);
    Assert.Equal(Money.USD(0m), finalAccount.Balance); // All withdrawals succeeded
    Assert.All(results, r => Assert.True(r.IsSuccess));
}
```

## Testing Async Streams (C# 8+)

```csharp
[Fact]
public async Task GetOrdersStreamAsync_ReturnsAllOrders()
{
    // Arrange
    var service = new OrderService(repository);
    await SeedOrders(5);
    var orders = new List<Order>();

    // Act
    await foreach (var order in service.GetOrdersStreamAsync())
    {
        orders.Add(order);
    }

    // Assert
    Assert.Equal(5, orders.Count);
}

[Fact]
public async Task GetOrdersStreamAsync_WithCancellation_StopsStreaming()
{
    // Arrange
    var service = new OrderService(repository);
    await SeedOrders(100);
    var cts = new CancellationTokenSource();
    var processedCount = 0;

    // Act
    try
    {
        await foreach (var order in service.GetOrdersStreamAsync(cts.Token))
        {
            processedCount++;
            if (processedCount == 10)
                cts.Cancel();
        }
    }
    catch (OperationCanceledException)
    {
        // Expected
    }

    // Assert
    Assert.Equal(10, processedCount);
}
```

## Testing Async Retry Logic

```csharp
[Fact]
public async Task ExecuteWithRetry_EventuallySucceeds()
{
    // Arrange
    var attemptCount = 0;
    var service = new RetryService();
    
    async Task<Result<string>> OperationThatFailsTwice()
    {
        attemptCount++;
        if (attemptCount < 3)
            return Result.Failure<string>("Temporary error");
        return Result.Success("Success");
    }

    // Act
    var result = await service.ExecuteWithRetryAsync(
        OperationThatFailsTwice,
        maxAttempts: 3
    );

    // Assert
    Assert.True(result.IsSuccess);
    Assert.Equal(3, attemptCount);
}
```

## Testing Async Event Handlers

```csharp
[Fact]
public async Task PublishEvent_TriggersAsyncHandlers()
{
    // Arrange
    var handlerCalled = false;
    var eventBus = new EventBus();
    
    eventBus.Subscribe<OrderCreatedEvent>(async evt =>
    {
        await Task.Delay(10); // Simulate async work
        handlerCalled = true;
    });

    // Act
    await eventBus.PublishAsync(new OrderCreatedEvent(OrderId.New()));
    await Task.Delay(100); // Give handlers time to complete

    // Assert
    Assert.True(handlerCalled);
}
```

## Testing with Fake Async Operations

```csharp
public class FakeAsyncEmailService : IEmailService
{
    public List<Email> SentEmails { get; } = new();

    public Task<Result<Unit>> SendAsync(
        EmailAddress to,
        string subject,
        string body)
    {
        // Simulate async without actual delay in tests
        SentEmails.Add(new Email(to, subject, body));
        return Task.FromResult(Result.Success(Unit.Value));
    }
}

[Fact]
public async Task ProcessOrder_SendsConfirmationEmail()
{
    // Arrange
    var emailService = new FakeAsyncEmailService();
    var service = new OrderService(repository, emailService);

    // Act
    await service.ProcessOrderAsync(orderId);

    // Assert
    Assert.Single(emailService.SentEmails);
    Assert.Contains("Confirmation", emailService.SentEmails[0].Subject);
}
```

## Common Pitfalls

### ❌ Async Void

```csharp
// ❌ Don't use async void in tests
[Fact]
public async void BadTest() // ❌ async void
{
    await SomeAsyncOperation();
}

// ✅ Use async Task
[Fact]
public async Task GoodTest() // ✅ async Task
{
    await SomeAsyncOperation();
}
```

### ❌ Not Awaiting

```csharp
// ❌ Forgotten await
[Fact]
public async Task BadTest()
{
    var result = repository.SaveAsync(order); // ❌ Not awaited
    Assert.True(result.IsSuccess); // Comparing Task to bool!
}

// ✅ Proper await
[Fact]
public async Task GoodTest()
{
    var result = await repository.SaveAsync(order); // ✅ Awaited
    Assert.True(result.IsSuccess);
}
```

## Guidelines

1. Always make test methods `async Task`
2. Always `await` async operations
3. Use `Task.WhenAll` for concurrent operations
4. Test cancellation paths explicitly
5. Use timeouts to catch hanging tests
6. Don't use `Task.Wait()` or `.Result` in tests

## See Also

- [Testing Exceptions](./testing-exceptions.md) — testing error conditions
- [Testing Side Effects](./testing-side-effects.md) — verifying async interactions
- [Integration Test Patterns](./integration-test-patterns.md) — testing real async I/O
- [Test Isolation and Independence](./test-isolation-independence.md) — async test isolation
