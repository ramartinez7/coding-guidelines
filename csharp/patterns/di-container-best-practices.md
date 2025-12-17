# DI Container Best Practices

> Use dependency injection containers effectively for loose coupling, testability, and proper lifetime management.

## Problem

Improper DI configuration leads to memory leaks, captive dependencies, and hard-to-debug runtime errors.

## Example

### ❌ Before

```csharp
public class OrderController : Controller
{
    public IActionResult PlaceOrder()
    {
        // Creating dependencies manually (tight coupling)
        var context = new AppDbContext();
        var repository = new OrderRepository(context);
        var service = new OrderService(repository);

        var result = service.PlaceOrder(order);
        return Ok(result);
    }
}

// Startup
services.AddSingleton<DbContext, AppDbContext>();  // WRONG lifetime!
services.AddSingleton<IOrderService, OrderService>();  // Leaks DbContext!
```

### ✅ After

```csharp
public class OrderController : Controller
{
    private readonly IOrderService orderService;

    public OrderController(IOrderService orderService)
    {
        this.orderService = orderService;
    }

    public IActionResult PlaceOrder()
    {
        var result = orderService.PlaceOrder(order);
        return Ok(result);
    }
}

// Startup
services.AddScoped<DbContext, AppDbContext>();
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<IOrderService, OrderService>();
```

## Best Practices

### 1. Use Correct Lifetimes

```csharp
// ✅ Correct lifetime registration
public void ConfigureServices(IServiceCollection services)
{
    // Transient: New instance per request
    services.AddTransient<IEmailSender, EmailSender>();
    services.AddTransient<IGuidGenerator, GuidGenerator>();

    // Scoped: One instance per HTTP request
    services.AddScoped<DbContext>();
    services.AddScoped<IOrderRepository, OrderRepository>();
    services.AddScoped<IUnitOfWork, UnitOfWork>();

    // Singleton: One instance for application lifetime
    services.AddSingleton<IConfiguration>(configuration);
    services.AddSingleton<IMemoryCache, MemoryCache>();
    services.AddSingleton<IHttpClientFactory, HttpClientFactory>();
}
```

### 2. Avoid Captive Dependencies

```csharp
// ❌ Captive dependency (scoped in singleton)
services.AddSingleton<IOrderService, OrderService>();  // Singleton
services.AddScoped<DbContext>();  // Scoped - will be captured!

public class OrderService : IOrderService
{
    private readonly DbContext context;  // Scoped captured by singleton!

    public OrderService(DbContext context)
    {
        this.context = context;  // Memory leak!
    }
}

// ✅ Match lifetimes appropriately
services.AddScoped<IOrderService, OrderService>();  // Match DbContext lifetime
services.AddScoped<DbContext>();
```

### 3. Use IServiceProvider Sparingly

```csharp
// ❌ Service locator anti-pattern
public class OrderService
{
    private readonly IServiceProvider serviceProvider;

    public OrderService(IServiceProvider serviceProvider)
    {
        this.serviceProvider = serviceProvider;
    }

    public void ProcessOrder(Order order)
    {
        var repository = serviceProvider.GetService<IOrderRepository>();
        // Hard to test, hidden dependencies
    }
}

// ✅ Explicit constructor injection
public class OrderService
{
    private readonly IOrderRepository repository;

    public OrderService(IOrderRepository repository)
    {
        this.repository = repository;
    }

    public void ProcessOrder(Order order)
    {
        repository.Save(order);
    }
}
```

### 4. Use Factory Pattern for Complex Creation

```csharp
// ✅ Factory for complex object creation
public interface IOrderProcessorFactory
{
    IOrderProcessor Create(OrderType orderType);
}

public class OrderProcessorFactory : IOrderProcessorFactory
{
    private readonly IServiceProvider serviceProvider;

    public OrderProcessorFactory(IServiceProvider serviceProvider)
    {
        this.serviceProvider = serviceProvider;
    }

    public IOrderProcessor Create(OrderType orderType)
    {
        return orderType switch
        {
            OrderType.Standard => serviceProvider.GetRequiredService<StandardOrderProcessor>(),
            OrderType.Express => serviceProvider.GetRequiredService<ExpressOrderProcessor>(),
            _ => throw new ArgumentException($"Unknown order type: {orderType}")
        };
    }
}
```

### 5. Register Implementations Explicitly

```csharp
// ❌ Auto-registration (magic, hard to understand)
services.Scan(scan => scan
    .FromAssemblyOf<Startup>()
    .AddClasses()
    .AsImplementedInterfaces()
    .WithScopedLifetime());

// ✅ Explicit registration (clear, searchable)
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<IOrderService, OrderService>();
services.AddScoped<IPaymentService, PaymentService>();
```

### 6. Use Options Pattern for Configuration

```csharp
// ✅ Options pattern
public class EmailSettings
{
    public string SmtpServer { get; set; }
    public int Port { get; set; }
    public string From { get; set; }
}

// Startup
services.Configure<EmailSettings>(
    configuration.GetSection("Email"));

// Usage
public class EmailSender
{
    private readonly EmailSettings settings;

    public EmailSender(IOptions<EmailSettings> options)
    {
        this.settings = options.Value;
    }
}
```

### 7. Validate Dependencies at Startup

```csharp
// ✅ Validate DI configuration
public void Configure(IApplicationBuilder app)
{
    // Validate all registered services can be resolved
    using (var scope = app.ApplicationServices.CreateScope())
    {
        var services = scope.ServiceProvider;
        try
        {
            services.GetRequiredService<IOrderService>();
            services.GetRequiredService<IPaymentService>();
            // etc.
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Service resolution failed at startup");
            throw;
        }
    }

    // Rest of configuration
}
```

### 8. Use Keyed Services (C# 12/.NET 8+)

```csharp
// ✅ Keyed services for multiple implementations
services.AddKeyedScoped<IPaymentProcessor, CreditCardProcessor>("CreditCard");
services.AddKeyedScoped<IPaymentProcessor, PayPalProcessor>("PayPal");

public class PaymentService
{
    private readonly IPaymentProcessor creditCardProcessor;
    private readonly IPaymentProcessor payPalProcessor;

    public PaymentService(
        [FromKeyedServices("CreditCard")] IPaymentProcessor creditCardProcessor,
        [FromKeyedServices("PayPal")] IPaymentProcessor payPalProcessor)
    {
        this.creditCardProcessor = creditCardProcessor;
        this.payPalProcessor = payPalProcessor;
    }
}
```

### 9. Avoid Constructor Over-Injection

```csharp
// ❌ Too many dependencies (code smell)
public class OrderService
{
    public OrderService(
        IOrderRepository orderRepository,
        IPaymentService paymentService,
        IEmailService emailService,
        IInventoryService inventoryService,
        IShippingService shippingService,
        ILoggingService loggingService,
        INotificationService notificationService,
        IAuditService auditService)  // 8+ dependencies!
    {
    }
}

// ✅ Refactor to smaller services or use facade
public class OrderService
{
    public OrderService(
        IOrderRepository repository,
        IOrderProcessor processor,  // Facade for payment, email, etc.
        ILogger<OrderService> logger)
    {
    }
}
```

### 10. Use IDisposable Properly

```csharp
// ✅ Container manages disposal
services.AddScoped<IOrderService, OrderService>();

public class OrderService : IOrderService, IDisposable
{
    private readonly FileStream logFile;

    public OrderService()
    {
        logFile = File.OpenWrite("orders.log");
    }

    public void Dispose()
    {
        logFile?.Dispose();
    }
}
// Container calls Dispose when scope ends
```

### 11. Use Scrutor for Decoration

```csharp
// ✅ Decorate services with cross-cutting concerns
services.AddScoped<IOrderService, OrderService>();

services.Decorate<IOrderService, CachedOrderService>();
services.Decorate<IOrderService, LoggingOrderService>();
services.Decorate<IOrderService, TransactionOrderService>();

// Results in: TransactionOrderService -> LoggingOrderService -> CachedOrderService -> OrderService
```

### 12. Test DI Configuration

```csharp
[Test]
public void ServiceConfiguration_AllServicesCanBeResolved()
{
    var services = new ServiceCollection();
    ConfigureServices(services);

    using var provider = services.BuildServiceProvider();

    // Verify all registered services can be resolved
    var orderService = provider.GetRequiredService<IOrderService>();
    var paymentService = provider.GetRequiredService<IPaymentService>();

    Assert.That(orderService, Is.Not.Null);
    Assert.That(paymentService, Is.Not.Null);
}
```

## Symptoms

- NullReferenceException at runtime from missing registrations
- Memory leaks from captive dependencies
- Slow startup from incorrect singleton usage
- Hard to test code with service locator
- Circular dependencies

## Benefits

- **Loose coupling** between components
- **Testability** with dependency injection
- **Proper resource management** with lifetimes
- **Flexibility** to swap implementations
- **Clear dependencies** in constructors

## See Also

- [Dependency Injection](./dependency-injection.md) — Core DI pattern
- [Type-Safe Dependency Injection](./type-safe-dependency-injection.md) — Compile-time validation
- [Composition Root](./composition-root.md) — DI configuration location
