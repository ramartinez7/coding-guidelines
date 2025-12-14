# Testing Side Effects

> Verify effects on the outside world—use test doubles to capture and verify interactions.

## Problem

Side effects like sending emails, logging, updating databases, or calling external APIs are hard to verify without test doubles. Tests that don't verify side effects miss important behaviors.

## Pattern

Use test doubles (spies, mocks, or fakes) to capture side effects and verify they occurred correctly.

## Example

### ❌ Before - Side Effects Not Verified

```csharp
[Fact]
public void ProcessOrder_WithValidOrder_Succeeds()
{
    // Arrange
    var service = new OrderService(repository, emailService, logger);
    var order = OrderBuilder.Default().Build();

    // Act
    service.Process(order.Id);

    // Assert
    Assert.Equal(OrderStatus.Processed, order.Status);
    // ❌ Email sent? Logged? We don't know!
}
```

### ✅ After - Side Effects Verified

```csharp
[Fact]
public void ProcessOrder_WithValidOrder_SendsEmailAndLogs()
{
    // Arrange
    var emailSpy = new SpyEmailService();
    var loggerSpy = new SpyLogger();
    var service = new OrderService(repository, emailSpy, loggerSpy);
    var order = OrderBuilder.Default().Build();

    // Act
    service.Process(order.Id);

    // Assert
    Assert.Equal(OrderStatus.Processed, order.Status);
    
    Assert.Single(emailSpy.SentEmails);
    Assert.Contains("Confirmation", emailSpy.SentEmails[0].Subject);
    
    Assert.Contains(loggerSpy.Messages, 
        m => m.Contains($"Processed order {order.Id}"));
}
```

## Testing Email Sending

### Using a Spy

```csharp
public class SpyEmailService : IEmailService
{
    public List<SentEmail> SentEmails { get; } = new();

    public Result<Unit> Send(EmailAddress to, string subject, string body)
    {
        SentEmails.Add(new SentEmail(to, subject, body));
        return Result.Success(Unit.Value);
    }
}

[Fact]
public void CreateOrder_SendsConfirmationEmail()
{
    // Arrange
    var emailSpy = new SpyEmailService();
    var service = new OrderService(repository, emailSpy);
    var customerEmail = EmailAddress.Create("customer@example.com").Value;

    // Act
    service.CreateOrder(customerId, items, customerEmail);

    // Assert
    Assert.Single(emailSpy.SentEmails);
    
    var email = emailSpy.SentEmails[0];
    Assert.Equal(customerEmail, email.To);
    Assert.Contains("Order Confirmation", email.Subject);
    Assert.Contains("Thank you", email.Body);
}
```

### Using a Mock (Moq)

```csharp
[Fact]
public void CreateOrder_SendsConfirmationEmail_UsingMock()
{
    // Arrange
    var emailMock = new Mock<IEmailService>();
    var service = new OrderService(repository, emailMock.Object);

    // Act
    service.CreateOrder(customerId, items, customerEmail);

    // Assert
    emailMock.Verify(
        x => x.Send(
            It.Is<EmailAddress>(e => e.ToString() == "customer@example.com"),
            It.Is<string>(s => s.Contains("Confirmation")),
            It.IsAny<string>()),
        Times.Once);
}
```

## Testing Logging

```csharp
public class SpyLogger : ILogger
{
    public List<LogEntry> Messages { get; } = new();

    public void LogInformation(string message)
    {
        Messages.Add(new LogEntry(LogLevel.Information, message));
    }

    public void LogError(string message, Exception? exception = null)
    {
        Messages.Add(new LogEntry(LogLevel.Error, message, exception));
    }
}

[Fact]
public void ProcessOrder_LogsProcessingSteps()
{
    // Arrange
    var logger = new SpyLogger();
    var service = new OrderService(repository, emailService, logger);

    // Act
    service.Process(orderId);

    // Assert
    Assert.Contains(logger.Messages,
        m => m.Level == LogLevel.Information && m.Message.Contains("Processing order"));
    
    Assert.Contains(logger.Messages,
        m => m.Level == LogLevel.Information && m.Message.Contains("Order processed successfully"));
}

[Fact]
public void ProcessOrder_WhenFails_LogsError()
{
    // Arrange
    var logger = new SpyLogger();
    var failingRepository = new FailingOrderRepository();
    var service = new OrderService(failingRepository, emailService, logger);

    // Act
    Assert.Throws<OrderProcessingException>(() =>
        service.Process(orderId));

    // Assert
    Assert.Contains(logger.Messages,
        m => m.Level == LogLevel.Error && m.Message.Contains("Failed to process"));
}
```

## Testing Database Operations

```csharp
[Fact]
public void SaveOrder_WritesToRepository()
{
    // Arrange
    var repository = new InMemoryOrderRepository();
    var service = new OrderService(repository);
    var order = OrderBuilder.Default().Build();

    // Act
    service.Save(order);

    // Assert
    var saved = repository.Get(order.Id);
    Assert.NotNull(saved);
    Assert.Equal(order.Id, saved.Id);
    Assert.Equal(order.Total, saved.Total);
}

[Fact]
public void UpdateOrder_ModifiesExistingRecord()
{
    // Arrange
    var repository = new InMemoryOrderRepository();
    var order = OrderBuilder.Default().Build();
    repository.Add(order);
    
    var service = new OrderService(repository);

    // Act
    order.UpdateTotal(Money.USD(200m));
    service.Update(order);

    // Assert
    var updated = repository.Get(order.Id);
    Assert.Equal(Money.USD(200m), updated.Total);
}
```

## Testing External API Calls

```csharp
public class SpyPaymentGateway : IPaymentGateway
{
    public List<PaymentRequest> ProcessedPayments { get; } = new();

    public Result<PaymentResult> ProcessPayment(PaymentRequest request)
    {
        ProcessedPayments.Add(request);
        return Result.Success(new PaymentResult(TransactionId.New(), "Approved"));
    }
}

[Fact]
public void ProcessOrder_ChargesPaymentGateway()
{
    // Arrange
    var paymentGateway = new SpyPaymentGateway();
    var service = new OrderService(repository, paymentGateway);
    var order = OrderBuilder.Default()
        .WithTotal(Money.USD(100m))
        .Build();

    // Act
    service.ProcessPayment(order.Id);

    // Assert
    Assert.Single(paymentGateway.ProcessedPayments);
    
    var payment = paymentGateway.ProcessedPayments[0];
    Assert.Equal(Money.USD(100m), payment.Amount);
    Assert.Equal(order.Id, payment.OrderId);
}
```

## Testing Event Publishing

```csharp
public class SpyEventBus : IEventBus
{
    public List<object> PublishedEvents { get; } = new();

    public void Publish<TEvent>(TEvent @event) where TEvent : class
    {
        PublishedEvents.Add(@event);
    }
}

[Fact]
public void CreateOrder_PublishesOrderCreatedEvent()
{
    // Arrange
    var eventBus = new SpyEventBus();
    var service = new OrderService(repository, eventBus);

    // Act
    var result = service.CreateOrder(customerId, items);

    // Assert
    Assert.Single(eventBus.PublishedEvents);
    
    var evt = Assert.IsType<OrderCreatedEvent>(eventBus.PublishedEvents[0]);
    Assert.Equal(result.Value.Id, evt.OrderId);
    Assert.Equal(customerId, evt.CustomerId);
}

[Fact]
public void ProcessOrder_PublishesOrderProcessedEvent()
{
    // Arrange
    var eventBus = new SpyEventBus();
    var service = new OrderService(repository, eventBus);
    var order = OrderBuilder.Default().Build();
    repository.Add(order);

    // Act
    service.Process(order.Id);

    // Assert
    var processedEvent = eventBus.PublishedEvents
        .OfType<OrderProcessedEvent>()
        .Single();
    
    Assert.Equal(order.Id, processedEvent.OrderId);
    Assert.Equal(OrderStatus.Processed, processedEvent.NewStatus);
}
```

## Testing File System Operations

```csharp
public class SpyFileSystem : IFileSystem
{
    public List<(string Path, string Content)> WrittenFiles { get; } = new();

    public void WriteAllText(string path, string content)
    {
        WrittenFiles.Add((path, content));
    }

    public string ReadAllText(string path)
    {
        var file = WrittenFiles.FirstOrDefault(f => f.Path == path);
        return file.Content ?? throw new FileNotFoundException();
    }
}

[Fact]
public void ExportOrder_WritesToFile()
{
    // Arrange
    var fileSystem = new SpyFileSystem();
    var exporter = new OrderExporter(fileSystem);
    var order = OrderBuilder.Default().Build();

    // Act
    exporter.Export(order, "/output/order.json");

    // Assert
    Assert.Single(fileSystem.WrittenFiles);
    
    var (path, content) = fileSystem.WrittenFiles[0];
    Assert.Equal("/output/order.json", path);
    Assert.Contains(order.Id.ToString(), content);
}
```

## Testing Multiple Side Effects

```csharp
[Fact]
public void ProcessOrder_PerformsAllSideEffects()
{
    // Arrange
    var repository = new InMemoryOrderRepository();
    var emailSpy = new SpyEmailService();
    var loggerSpy = new SpyLogger();
    var eventBus = new SpyEventBus();
    
    var service = new OrderService(repository, emailSpy, loggerSpy, eventBus);
    var order = OrderBuilder.Default().Build();
    repository.Add(order);

    // Act
    service.Process(order.Id);

    // Assert - Repository
    var updated = repository.Get(order.Id);
    Assert.Equal(OrderStatus.Processed, updated.Status);
    
    // Assert - Email
    Assert.Single(emailSpy.SentEmails);
    Assert.Contains("Confirmation", emailSpy.SentEmails[0].Subject);
    
    // Assert - Logging
    Assert.Contains(loggerSpy.Messages, m => m.Message.Contains("Processed"));
    
    // Assert - Events
    Assert.Contains(eventBus.PublishedEvents, e => e is OrderProcessedEvent);
}
```

## Testing Side Effect Order

```csharp
[Fact]
public void ProcessOrder_PerformsSideEffectsInCorrectOrder()
{
    // Arrange
    var operations = new List<string>();
    
    var repository = new SpyOrderRepository(operations);
    var emailService = new SpyEmailService(operations);
    var eventBus = new SpyEventBus(operations);
    
    var service = new OrderService(repository, emailService, eventBus);

    // Act
    service.Process(orderId);

    // Assert
    Assert.Equal(new[] { "SaveOrder", "SendEmail", "PublishEvent" }, operations);
}
```

## Testing Conditional Side Effects

```csharp
[Fact]
public void ProcessOrder_ForStandardCustomer_DoesNotSendPremiumEmail()
{
    // Arrange
    var emailSpy = new SpyEmailService();
    var customer = CustomerMother.StandardCustomer();
    var service = new OrderService(repository, emailSpy);

    // Act
    service.Process(orderId);

    // Assert
    Assert.DoesNotContain(emailSpy.SentEmails,
        e => e.Subject.Contains("Premium"));
}

[Fact]
public void ProcessOrder_ForPremiumCustomer_SendsPremiumEmail()
{
    // Arrange
    var emailSpy = new SpyEmailService();
    var customer = CustomerMother.PremiumCustomer();
    var service = new OrderService(repository, emailSpy);

    // Act
    service.Process(orderId);

    // Assert
    Assert.Contains(emailSpy.SentEmails,
        e => e.Subject.Contains("Premium"));
}
```

## Guidelines

1. **Use spies** for capturing interactions
2. **Use mocks** for complex behavior verification
3. **Verify side effects** explicitly, don't assume
4. **Test negative cases** (side effects that shouldn't happen)
5. **Keep test doubles simple** and focused

## Benefits

1. **Verification**: Ensures all intended effects occur
2. **Isolation**: Tests don't require real email servers, databases
3. **Speed**: In-memory test doubles are fast
4. **Reliability**: No external dependencies to fail
5. **Completeness**: Tests cover the full behavior

## See Also

- [Test Doubles](./test-doubles.md) — types of test doubles
- [Integration Test Patterns](./integration-test-patterns.md) — testing with real side effects
- [Testing Async Code](./testing-async-code.md) — async side effects
- [Test Isolation and Independence](./test-isolation-independence.md) — isolating side effects
