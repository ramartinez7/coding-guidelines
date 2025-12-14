# Test Doubles: Mocks, Stubs, Fakes

> Understand when to use mocks (behavior verification), stubs (state setup), and fakes (working implementations).

## Problem

Confusion about test doubles leads to brittle tests, over-mocking, and tests that don't catch real bugs. Each type of test double has specific use cases and trade-offs.

## Types of Test Doubles

### 1. Dummy

Objects passed but never used—just to satisfy parameter lists.

```csharp
[Fact]
public void CreateOrder_WithValidInput_Succeeds()
{
    // Arrange
    var dummyLogger = new DummyLogger(); // Never called
    var service = new OrderService(repository, dummyLogger);

    // Act
    var result = service.CreateOrder(customerId, items);

    // Assert
    Assert.True(result.IsSuccess);
}

public class DummyLogger : ILogger
{
    public void Log(string message) { } // Do nothing
}
```

### 2. Stub

Provides canned responses to calls—sets up state for the test.

```csharp
[Fact]
public void CalculateDiscount_ForPremiumCustomer_AppliesPremiumRate()
{
    // Arrange
    var customerStub = new StubCustomerRepository();
    customerStub.SetCustomerTier(customerId, CustomerTier.Premium);
    
    var service = new DiscountService(customerStub);

    // Act
    var discount = service.CalculateDiscount(customerId, Money.USD(100m));

    // Assert
    Assert.Equal(Money.USD(20m), discount); // 20% premium discount
}

public class StubCustomerRepository : ICustomerRepository
{
    private readonly Dictionary<CustomerId, CustomerTier> tiers = new();

    public void SetCustomerTier(CustomerId id, CustomerTier tier)
    {
        tiers[id] = tier;
    }

    public CustomerTier GetTier(CustomerId id) => tiers.GetValueOrDefault(id);
}
```

### 3. Fake

Working implementation with shortcuts—simpler than production.

```csharp
[Fact]
public void SaveOrder_WithValidOrder_CanBeRetrieved()
{
    // Arrange
    var fakeRepository = new InMemoryOrderRepository();
    var order = Order.Create(OrderId.New(), CustomerId.New(), Money.USD(100m));

    // Act
    fakeRepository.Save(order);
    var retrieved = fakeRepository.Get(order.Id);

    // Assert
    Assert.NotNull(retrieved);
    Assert.Equal(order.Id, retrieved.Id);
}

public class InMemoryOrderRepository : IOrderRepository
{
    private readonly Dictionary<OrderId, Order> orders = new();

    public void Save(Order order)
    {
        orders[order.Id] = order;
    }

    public Order? Get(OrderId id)
    {
        orders.TryGetValue(id, out var order);
        return order;
    }

    public IReadOnlyList<Order> GetAll() => orders.Values.ToList();
}
```

### 4. Spy

Records information about calls—used for behavior verification.

```csharp
[Fact]
public void ProcessOrder_WithValidOrder_LogsProcessing()
{
    // Arrange
    var loggerSpy = new SpyLogger();
    var service = new OrderService(repository, loggerSpy);

    // Act
    service.ProcessOrder(orderId);

    // Assert
    Assert.Single(loggerSpy.LoggedMessages);
    Assert.Contains("Processing order", loggerSpy.LoggedMessages[0]);
}

public class SpyLogger : ILogger
{
    public List<string> LoggedMessages { get; } = new();

    public void Log(string message)
    {
        LoggedMessages.Add(message);
    }
}
```

### 5. Mock

Pre-programmed expectations—verifies behavior.

```csharp
[Fact]
public void ProcessOrder_WithValidOrder_SendsConfirmationEmail()
{
    // Arrange
    var emailMock = new Mock<IEmailService>();
    var service = new OrderService(repository, emailMock.Object);

    // Act
    service.ProcessOrder(orderId);

    // Assert
    emailMock.Verify(
        x => x.SendEmail(
            It.Is<string>(email => email.Contains("@")),
            It.Is<string>(subject => subject.Contains("Confirmation")),
            It.IsAny<string>()),
        Times.Once);
}
```

## When to Use Each Type

### Use Stubs When:
- You need to control indirect inputs
- You're setting up test state
- You're testing the system's response to specific conditions

```csharp
// Testing how the system handles different customer tiers
var customerStub = new StubCustomerRepository();
customerStub.SetCustomerTier(customerId, CustomerTier.VIP);

var discount = service.CalculateDiscount(customerId, amount);
Assert.Equal(expectedVIPDiscount, discount);
```

### Use Mocks When:
- You need to verify interactions
- You're testing that the system calls dependencies correctly
- The behavior itself is what matters, not the return value

```csharp
// Testing that the system sends audit logs
var auditMock = new Mock<IAuditService>();

service.ProcessSensitiveOperation(userId, data);

auditMock.Verify(x => x.LogAccess(userId, It.IsAny<DateTime>()), Times.Once);
```

### Use Fakes When:
- You need realistic behavior
- Multiple tests share the same dependency
- You want fast, in-memory implementations

```csharp
// Shared fake used across many tests
public class InMemoryOrderRepository : IOrderRepository
{
    // Full working implementation using Dictionary
}
```

## Hand-Rolled vs. Framework Mocks

### Hand-Rolled (Recommended for Fakes and Stubs)

```csharp
public class InMemoryEmailService : IEmailService
{
    public List<Email> SentEmails { get; } = new();

    public Result<Unit> SendEmail(string to, string subject, string body)
    {
        SentEmails.Add(new Email(to, subject, body));
        return Result.Success(Unit.Value);
    }
}

// Clear, reusable, easy to debug
[Fact]
public void Test()
{
    var emailService = new InMemoryEmailService();
    service.ProcessOrder(orderId);
    
    Assert.Single(emailService.SentEmails);
    Assert.Equal(expectedEmail, emailService.SentEmails[0].To);
}
```

### Framework Mocks (Good for Complex Behavior Verification)

```csharp
[Fact]
public void ProcessOrder_CallsDependenciesInCorrectOrder()
{
    var sequence = new MockSequence();
    
    var inventoryMock = new Mock<IInventoryService>(MockBehavior.Strict);
    var paymentMock = new Mock<IPaymentService>(MockBehavior.Strict);
    var shippingMock = new Mock<IShippingService>(MockBehavior.Strict);
    
    // Verify specific call order
    inventoryMock.InSequence(sequence)
        .Setup(x => x.ReserveItems(It.IsAny<OrderId>()))
        .Returns(Result.Success(Unit.Value));
    
    paymentMock.InSequence(sequence)
        .Setup(x => x.ProcessPayment(It.IsAny<OrderId>()))
        .Returns(Result.Success(Unit.Value));
    
    shippingMock.InSequence(sequence)
        .Setup(x => x.CreateShipment(It.IsAny<OrderId>()))
        .Returns(Result.Success(Unit.Value));
    
    var service = new OrderService(inventoryMock.Object, paymentMock.Object, shippingMock.Object);
    
    service.ProcessOrder(orderId);
    
    inventoryMock.VerifyAll();
    paymentMock.VerifyAll();
    shippingMock.VerifyAll();
}
```

## Anti-Patterns

### Over-Mocking

```csharp
// ❌ Mocking too much
[Fact]
public void Test()
{
    var mock1 = new Mock<IDependency1>();
    var mock2 = new Mock<IDependency2>();
    var mock3 = new Mock<IDependency3>();
    var mock4 = new Mock<IDependency4>();
    var mock5 = new Mock<IDependency5>();
    
    // If you need this many mocks, the class may have too many dependencies
}

// ✅ Use real objects when possible
[Fact]
public void Test()
{
    var realService = new ActualDependency(); // No mocking needed
    var fake = new InMemoryRepository();
    
    var sut = new SystemUnderTest(realService, fake);
}
```

### Fragile Mocks

```csharp
// ❌ Too specific, breaks with refactoring
mock.Verify(x => x.Process(
    It.Is<Order>(o => o.Id == orderId && 
                      o.Total == Money.USD(100m) &&
                      o.Status == OrderStatus.Pending)),
    Times.Once);

// ✅ Verify what matters
mock.Verify(x => x.Process(It.Is<Order>(o => o.Id == orderId)), Times.Once);
```

### Mocking Value Objects

```csharp
// ❌ Don't mock value objects
var mock = new Mock<IEmailAddress>();

// ✅ Use real value objects
var email = EmailAddress.Create("test@example.com");
```

## Guidelines

1. **Prefer fakes for repositories and simple dependencies**
2. **Use stubs to control test inputs**
3. **Use mocks sparingly, only for behavior verification**
4. **Don't mock what you don't own** (third-party libraries)
5. **Keep test doubles simple**—complex doubles indicate design issues

## Benefits

1. **Speed**: Tests run in memory without I/O
2. **Isolation**: Test one component at a time
3. **Control**: Simulate edge cases and error conditions
4. **Determinism**: No external dependencies means reliable tests

## See Also

- [Testing Side Effects](./testing-side-effects.md) — verifying interactions
- [Integration Test Patterns](./integration-test-patterns.md) — testing with real dependencies
- [Test Isolation and Independence](./test-isolation-independence.md) — using doubles for isolation
- [Testing Domain Invariants](./testing-domain-invariants.md) — testing business logic
