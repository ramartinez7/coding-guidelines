# Dependency Injection

> Inject dependencies through constructors rather than creating them internally or using service locators.

## Problem

When classes create their own dependencies or use service locators, they become tightly coupled, hard to test, and inflexible. Changes to dependencies ripple throughout the codebase.

## Example

### ❌ Before

```csharp
public class OrderService
{
    private readonly OrderRepository repository;
    private readonly EmailService emailService;
    private readonly Logger logger;

    public OrderService()
    {
        // Tight coupling to concrete implementations
        this.repository = new OrderRepository();
        this.emailService = new EmailService();
        this.logger = new Logger();
    }

    public void ProcessOrder(Order order)
    {
        repository.Save(order);
        emailService.SendConfirmation(order);
        logger.Log($"Processed order {order.Id}");
    }
}

// Using Service Locator (anti-pattern)
public class OrderServiceWithLocator
{
    public void ProcessOrder(Order order)
    {
        var repository = ServiceLocator.Get<OrderRepository>();
        var emailService = ServiceLocator.Get<EmailService>();
        var logger = ServiceLocator.Get<Logger>();

        repository.Save(order);
        emailService.SendConfirmation(order);
        logger.Log($"Processed order {order.Id}");
    }
}
```

**Problems:**
- Cannot test without real database and email server
- Cannot swap implementations (e.g., for different environments)
- Hidden dependencies (not visible in constructor)
- Difficult to identify circular dependencies
- Service locator adds global state and runtime errors

### ✅ After

```csharp
// Define abstractions
public interface IOrderRepository
{
    void Save(Order order);
}

public interface IEmailService
{
    void SendConfirmation(Order order);
}

public interface ILogger
{
    void Log(string message);
}

// Inject dependencies through constructor
public class OrderService
{
    private readonly IOrderRepository repository;
    private readonly IEmailService emailService;
    private readonly ILogger logger;

    public OrderService(
        IOrderRepository repository,
        IEmailService emailService,
        ILogger logger)
    {
        this.repository = repository;
        this.emailService = emailService;
        this.logger = logger;
    }

    public void ProcessOrder(Order order)
    {
        repository.Save(order);
        emailService.SendConfirmation(order);
        logger.Log($"Processed order {order.Id}");
    }
}

// Configure DI container at application startup
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddScoped<IOrderRepository, SqlOrderRepository>();
        services.AddScoped<IEmailService, SmtpEmailService>();
        services.AddSingleton<ILogger, FileLogger>();
        services.AddScoped<OrderService>();
    }
}
```

## Constructor Injection

The most common and recommended form of dependency injection.

### Benefits

```csharp
public class PaymentProcessor
{
    private readonly IPaymentGateway gateway;
    private readonly ILogger logger;

    // Dependencies are explicit and required
    public PaymentProcessor(IPaymentGateway gateway, ILogger logger)
    {
        this.gateway = gateway ?? throw new ArgumentNullException(nameof(gateway));
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    public Result<Payment, PaymentError> ProcessPayment(Order order)
    {
        logger.Log($"Processing payment for order {order.Id}");
        return gateway.Charge(order.Total, order.PaymentMethod);
    }
}
```

**Advantages:**
- Dependencies are explicit and visible
- Class is immutable after construction
- Impossible to use class without required dependencies
- Easy to test with mocks or stubs

## Method Injection

Use when a dependency is only needed for specific operations.

```csharp
public class ReportGenerator
{
    private readonly IReportRepository repository;

    public ReportGenerator(IReportRepository repository)
    {
        this.repository = repository;
    }

    // Formatter injected per operation (different formats for different reports)
    public string GenerateReport(ReportId id, IReportFormatter formatter)
    {
        var data = repository.GetReportData(id);
        return formatter.Format(data);
    }
}

// Usage
var pdfFormatter = new PdfReportFormatter();
var csvFormatter = new CsvReportFormatter();

reportGenerator.GenerateReport(reportId, pdfFormatter);
reportGenerator.GenerateReport(reportId, csvFormatter);
```

## Property Injection (Use Sparingly)

Only for optional dependencies with sensible defaults.

```csharp
public class UserService
{
    private readonly IUserRepository repository;

    public UserService(IUserRepository repository)
    {
        this.repository = repository;
    }

    // Optional: can override for testing or special cases
    public ILogger Logger { get; set; } = NullLogger.Instance;

    public void CreateUser(User user)
    {
        repository.Save(user);
        Logger.Log($"Created user {user.Id}");
    }
}
```

**Warning:** Property injection makes dependencies less visible. Prefer constructor injection.

## Lifetime Management

### Transient

New instance created each time it's requested.

```csharp
// Each request gets a new instance
services.AddTransient<IEmailService, SmtpEmailService>();
```

**Use for:** Lightweight, stateless services.

### Scoped

One instance per request/scope.

```csharp
// Same instance within a single HTTP request
services.AddScoped<IOrderRepository, SqlOrderRepository>();
services.AddScoped<IUnitOfWork, UnitOfWork>();
```

**Use for:** Database connections, unit of work, per-request state.

### Singleton

Single instance for the application lifetime.

```csharp
// One instance shared across all requests
services.AddSingleton<ILogger, FileLogger>();
services.AddSingleton<ICache, MemoryCache>();
```

**Use for:** Thread-safe, stateless services, caches, configuration.

**Warning:** Be careful with singletons—they must be thread-safe.

## Testing with Dependency Injection

### Manual Injection (Simple Tests)

```csharp
public class OrderServiceTests
{
    [Fact]
    public void ProcessOrder_SavesOrder()
    {
        // Arrange
        var mockRepository = new MockOrderRepository();
        var mockEmailService = new MockEmailService();
        var mockLogger = new MockLogger();

        var service = new OrderService(mockRepository, mockEmailService, mockLogger);
        var order = new Order(/* ... */);

        // Act
        service.ProcessOrder(order);

        // Assert
        Assert.True(mockRepository.WasSaveCalled);
    }
}
```

### Using Mocking Frameworks

```csharp
public class OrderServiceTests
{
    [Fact]
    public void ProcessOrder_LogsOrderId()
    {
        // Arrange
        var mockRepository = Substitute.For<IOrderRepository>();
        var mockEmailService = Substitute.For<IEmailService>();
        var mockLogger = Substitute.For<ILogger>();

        var service = new OrderService(mockRepository, mockEmailService, mockLogger);
        var order = new Order(OrderId.New(), /* ... */);

        // Act
        service.ProcessOrder(order);

        // Assert
        mockLogger.Received(1).Log(Arg.Is<string>(s => s.Contains(order.Id.ToString())));
    }
}
```

## Avoiding Anti-Patterns

### ❌ Service Locator

```csharp
// Anti-pattern: hidden dependencies
public class OrderService
{
    public void ProcessOrder(Order order)
    {
        var repository = ServiceLocator.Get<IOrderRepository>();
        repository.Save(order);
    }
}
```

**Why it's bad:**
- Dependencies are hidden
- Runtime errors when service not registered
- Global state makes testing harder
- Difficult to track what depends on what

### ❌ New Keyword for Dependencies

```csharp
// Anti-pattern: tight coupling
public class OrderService
{
    private readonly OrderRepository repository = new OrderRepository();
}
```

**Why it's bad:**
- Cannot test with mocks
- Cannot swap implementations
- Changes to OrderRepository constructor break OrderService

### ❌ Injecting Too Many Dependencies

```csharp
// Code smell: too many dependencies
public class OrderService
{
    public OrderService(
        IOrderRepository orderRepo,
        ICustomerRepository customerRepo,
        IProductRepository productRepo,
        IInventoryService inventoryService,
        IPaymentGateway paymentGateway,
        IShippingService shippingService,
        IEmailService emailService,
        INotificationService notificationService,
        ILogger logger,
        ICache cache,
        IEventBus eventBus)
    {
        // Too many dependencies suggests violation of Single Responsibility Principle
    }
}
```

**Solution:** Break the class into smaller, focused classes, or use facade pattern.

## Best Practices

### Prefer Interfaces Over Abstract Classes

```csharp
// ✅ Interface allows multiple implementations
public interface IPaymentGateway
{
    Result<Payment, PaymentError> ProcessPayment(Money amount);
}

// ✅ Can implement multiple interfaces
public class StripeGateway : IPaymentGateway, IRefundService
{
    // Implementation
}
```

### Keep Constructors Simple

```csharp
// ❌ Constructor does work
public class OrderService
{
    public OrderService(IOrderRepository repository)
    {
        this.repository = repository;
        this.repository.Initialize();  // Bad: doing work in constructor
        this.cache = LoadCache();      // Bad: I/O in constructor
    }
}

// ✅ Constructor only assigns dependencies
public class OrderService
{
    private readonly IOrderRepository repository;

    public OrderService(IOrderRepository repository)
    {
        this.repository = repository ?? throw new ArgumentNullException(nameof(repository));
    }

    public void Initialize()
    {
        repository.Initialize();
    }
}
```

### Use Factory Pattern for Complex Creation

```csharp
public interface IPaymentProcessorFactory
{
    IPaymentProcessor Create(PaymentMethod method);
}

public class PaymentProcessorFactory : IPaymentProcessorFactory
{
    private readonly IServiceProvider serviceProvider;

    public PaymentProcessorFactory(IServiceProvider serviceProvider)
    {
        this.serviceProvider = serviceProvider;
    }

    public IPaymentProcessor Create(PaymentMethod method)
    {
        return method switch
        {
            PaymentMethod.CreditCard => serviceProvider.GetService<CreditCardProcessor>(),
            PaymentMethod.PayPal => serviceProvider.GetService<PayPalProcessor>(),
            PaymentMethod.BankTransfer => serviceProvider.GetService<BankTransferProcessor>(),
            _ => throw new ArgumentException($"Unknown payment method: {method}")
        };
    }
}
```

## Symptoms

- Difficulty writing unit tests (need real database, file system, etc.)
- Cannot run tests in parallel (shared global state)
- Hard to swap implementations (e.g., for different environments)
- Hidden dependencies discovered at runtime
- Circular dependency errors
- Changes to one class require changes to many others

## Benefits

- **Testability**: Easy to inject mocks and stubs
- **Flexibility**: Swap implementations without changing code
- **Maintainability**: Dependencies are explicit and visible
- **Separation of Concerns**: Classes focus on business logic, not object creation
- **Reusability**: Classes can be used with different implementations

## See Also

- [SOLID Principles](./solid-principles.md) — Dependency Inversion Principle
- [Repository Pattern](./repository-pattern.md) — Injecting data access dependencies
- [Capability Security](./capability-security.md) — Injecting authorization tokens
