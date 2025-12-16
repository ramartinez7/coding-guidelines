# Port-Adapter Separation (Hexagonal Core)

> Core business logic defines abstract ports (interfaces), while adapters implement concrete connections to external systems—enabling the domain to remain independent of technical details.

## Problem

Applications often tightly couple business logic to specific technologies (databases, message queues, external APIs). When technology choices change, the business logic must be rewritten. The hexagonal architecture pattern solves this by defining clear ports (interfaces) in the core and pushing all technology-specific code to adapters.

## Core Concepts

In hexagonal architecture:
- **Ports** are interfaces defined by the domain/application layer
- **Adapters** are concrete implementations that live in the infrastructure layer
- **Primary (Driving) Adapters** initiate actions (HTTP controllers, CLI commands, message consumers)
- **Secondary (Driven) Adapters** are called by the application (database repositories, email services, external APIs)

```
┌─────────────────────────────────────────────┐
│         Primary Adapters (Driving)          │
│      (HTTP, CLI, gRPC, Message Queue)       │
└──────────────────┬──────────────────────────┘
                   │
         ┌─────────▼─────────┐
         │   Primary Ports   │ (Use Cases, Commands)
         │                   │
         │  ┌─────────────┐  │
         │  │   Domain    │  │
         │  │    Logic    │  │
         │  └─────────────┘  │
         │                   │
         │  Secondary Ports  │ (IRepository, IEmailService)
         └─────────┬─────────┘
                   │
┌──────────────────▼──────────────────────────┐
│        Secondary Adapters (Driven)          │
│   (Database, Email, External APIs, Cache)   │
└─────────────────────────────────────────────┘
```

## Example

### ❌ Before (Tight Coupling to Infrastructure)

```csharp
// Business logic directly depends on specific technologies
public class OrderService
{
    private readonly SqlConnection _connection;
    private readonly SmtpClient _smtpClient;
    private readonly RestClient _restClient;
    
    public async Task ProcessOrder(int orderId)
    {
        // SQL Server specific code in business logic
        var command = new SqlCommand(
            "SELECT * FROM Orders WHERE Id = @Id", 
            _connection);
        command.Parameters.AddWithValue("@Id", orderId);
        
        using var reader = await command.ExecuteReaderAsync();
        // ... process order
        
        // SMTP specific code
        var message = new MailMessage
        {
            To = { "customer@example.com" },
            Subject = "Order Confirmation"
        };
        await _smtpClient.SendMailAsync(message);
        
        // REST client specific code
        var request = new RestRequest("/inventory/reserve", Method.Post);
        await _restClient.ExecuteAsync(request);
    }
}
```

**Problems:**
- Business logic knows about SQL Server, SMTP, REST
- Can't swap technologies without rewriting business logic
- Hard to test without real database, email server, and external API
- Violates Single Responsibility Principle (business logic + infrastructure)

### ✅ After (Port-Adapter Separation)

#### 1. Domain Layer (Core - Defines Ports)

```csharp
// Domain/Entities/Order.cs
namespace MyApp.Domain.Entities
{
    /// <summary>
    /// Pure domain entity - no infrastructure dependencies.
    /// </summary>
    public sealed class Order
    {
        public OrderId Id { get; }
        public CustomerId CustomerId { get; }
        public Money Total { get; private set; }
        public OrderStatus Status { get; private set; }
        
        private Order(OrderId id, CustomerId customerId)
        {
            Id = id;
            CustomerId = customerId;
            Status = OrderStatus.Pending;
        }
        
        public static Order Create(CustomerId customerId)
        {
            return new Order(OrderId.New(), customerId);
        }
        
        public Result<Unit, string> Submit()
        {
            if (Status != OrderStatus.Pending)
                return Result<Unit, string>.Failure("Order already submitted");
            
            Status = OrderStatus.Submitted;
            return Result<Unit, string>.Success(Unit.Value);
        }
    }
}

// Domain/Ports/IOrderRepository.cs (Secondary Port - Driven)
namespace MyApp.Domain.Ports
{
    /// <summary>
    /// Port defined by domain - no knowledge of database technology.
    /// This is what the domain NEEDS, not how it's implemented.
    /// </summary>
    public interface IOrderRepository
    {
        Task<Option<Order>> GetByIdAsync(OrderId id);
        Task SaveAsync(Order order);
    }
}

// Domain/Ports/IInventoryService.cs (Secondary Port - Driven)
namespace MyApp.Domain.Ports
{
    /// <summary>
    /// Port for external inventory system - domain doesn't know
    /// if it's REST, gRPC, message queue, or in-memory.
    /// </summary>
    public interface IInventoryService
    {
        Task<Result<Unit, string>> ReserveStockAsync(ProductId productId, int quantity);
    }
}
```

#### 2. Application Layer (Use Cases - Primary Ports)

```csharp
// Application/UseCases/ProcessOrder/ProcessOrderCommand.cs
namespace MyApp.Application.UseCases.ProcessOrder
{
    /// <summary>
    /// Primary Port (Input) - defines what the application can do.
    /// This is the "driving" side of the hexagon.
    /// </summary>
    public sealed record ProcessOrderCommand(OrderId OrderId);
    
    /// <summary>
    /// Use case orchestrates domain logic and calls secondary ports.
    /// </summary>
    public sealed class ProcessOrderUseCase
    {
        // Dependencies are all secondary ports (driven)
        private readonly IOrderRepository _orderRepository;
        private readonly IEmailService _emailService;
        private readonly IInventoryService _inventoryService;
        private readonly IUnitOfWork _unitOfWork;
        
        public ProcessOrderUseCase(
            IOrderRepository orderRepository,
            IEmailService emailService,
            IInventoryService inventoryService,
            IUnitOfWork unitOfWork)
        {
            _orderRepository = orderRepository;
            _emailService = emailService;
            _inventoryService = inventoryService;
            _unitOfWork = unitOfWork;
        }
        
        public async Task<Result<Unit, ProcessOrderError>> ExecuteAsync(
            ProcessOrderCommand command)
        {
            // Get order through port
            var orderResult = await _orderRepository.GetByIdAsync(command.OrderId);
            
            var order = await orderResult.Match(
                onSome: o => Task.FromResult(o),
                onNone: () => Task.FromResult<Order?>(null));
            
            if (order == null)
                return Result<Unit, ProcessOrderError>.Failure(
                    new ProcessOrderError.OrderNotFound(command.OrderId));
            
            // Domain logic
            var submitResult = order.Submit();
            if (!submitResult.IsSuccess)
                return Result<Unit, ProcessOrderError>.Failure(
                    new ProcessOrderError.InvalidState(submitResult.Error));
            
            // Reserve inventory through port
            foreach (var item in order.Items)
            {
                var reserveResult = await _inventoryService.ReserveStockAsync(
                    item.ProductId, 
                    item.Quantity);
                
                if (!reserveResult.IsSuccess)
                    return Result<Unit, ProcessOrderError>.Failure(
                        new ProcessOrderError.InsufficientStock(item.ProductId));
            }
            
            // Persist through port
            await _orderRepository.SaveAsync(order);
            await _unitOfWork.CommitAsync();
            
            // Send notification through port
            await _emailService.SendOrderConfirmationAsync(order.Id, order.CustomerId);
            
            return Result<Unit, ProcessOrderError>.Success(Unit.Value);
        }
    }
    
    public abstract record ProcessOrderError
    {
        public sealed record OrderNotFound(OrderId OrderId) : ProcessOrderError;
        public sealed record InvalidState(string Reason) : ProcessOrderError;
        public sealed record InsufficientStock(ProductId ProductId) : ProcessOrderError;
    }
}

// Application/Ports/IEmailService.cs (Secondary Port - Driven)
namespace MyApp.Application.Ports
{
    /// <summary>
    /// Application-level port for email service.
    /// </summary>
    public interface IEmailService
    {
        Task SendOrderConfirmationAsync(OrderId orderId, CustomerId customerId);
    }
}
```

#### 3. Infrastructure Layer (Secondary Adapters - Driven)

```csharp
// Infrastructure/Adapters/Persistence/SqlOrderRepository.cs
namespace MyApp.Infrastructure.Adapters.Persistence
{
    /// <summary>
    /// Adapter that implements the domain port using SQL Server.
    /// Can be swapped for PostgreSQL, MongoDB, etc. without changing domain.
    /// </summary>
    public sealed class SqlOrderRepository : IOrderRepository
    {
        private readonly IDbConnection _connection;
        
        public SqlOrderRepository(IDbConnection connection)
        {
            _connection = connection;
        }
        
        public async Task<Option<Order>> GetByIdAsync(OrderId id)
        {
            var sql = "SELECT * FROM Orders WHERE Id = @Id";
            var entity = await _connection.QueryFirstOrDefaultAsync<OrderEntity>(
                sql, 
                new { Id = id.Value });
            
            if (entity == null)
                return Option<Order>.None;
            
            // Translate from persistence model to domain model
            var order = MapToDomain(entity);
            return Option<Order>.Some(order);
        }
        
        public async Task SaveAsync(Order order)
        {
            var entity = MapToEntity(order);
            var sql = @"
                UPDATE Orders 
                SET Status = @Status, Total = @Total 
                WHERE Id = @Id";
            
            await _connection.ExecuteAsync(sql, entity);
        }
        
        private Order MapToDomain(OrderEntity entity) { /* ... */ }
        private OrderEntity MapToEntity(Order order) { /* ... */ }
    }
}

// Infrastructure/Adapters/Email/SendGridEmailAdapter.cs
namespace MyApp.Infrastructure.Adapters.Email
{
    /// <summary>
    /// Adapter that implements email port using SendGrid.
    /// Can be swapped for SMTP, AWS SES, etc. without changing application.
    /// </summary>
    public sealed class SendGridEmailAdapter : IEmailService
    {
        private readonly ISendGridClient _sendGridClient;
        private readonly ICustomerRepository _customerRepository;
        
        public SendGridEmailAdapter(
            ISendGridClient sendGridClient,
            ICustomerRepository customerRepository)
        {
            _sendGridClient = sendGridClient;
            _customerRepository = customerRepository;
        }
        
        public async Task SendOrderConfirmationAsync(OrderId orderId, CustomerId customerId)
        {
            var customer = await _customerRepository.GetByIdAsync(customerId);
            
            await customer.Match(
                onSome: async c =>
                {
                    var message = new SendGridMessage
                    {
                        Subject = "Order Confirmation",
                        PlainTextContent = $"Your order {orderId.Value} has been confirmed.",
                        To = new List<EmailAddress> { new(c.Email.Value) }
                    };
                    
                    await _sendGridClient.SendEmailAsync(message);
                },
                onNone: () => Task.CompletedTask);
        }
    }
}

// Infrastructure/Adapters/Inventory/RestInventoryAdapter.cs
namespace MyApp.Infrastructure.Adapters.Inventory
{
    /// <summary>
    /// Adapter that implements inventory port using REST API.
    /// Can be swapped for gRPC, message queue, etc. without changing domain.
    /// </summary>
    public sealed class RestInventoryAdapter : IInventoryService
    {
        private readonly HttpClient _httpClient;
        
        public RestInventoryAdapter(HttpClient httpClient)
        {
            _httpClient = httpClient;
        }
        
        public async Task<Result<Unit, string>> ReserveStockAsync(
            ProductId productId, 
            int quantity)
        {
            var request = new
            {
                ProductId = productId.Value,
                Quantity = quantity
            };
            
            var response = await _httpClient.PostAsJsonAsync(
                "/api/inventory/reserve", 
                request);
            
            if (response.IsSuccessStatusCode)
                return Result<Unit, string>.Success(Unit.Value);
            
            var error = await response.Content.ReadAsStringAsync();
            return Result<Unit, string>.Failure(error);
        }
    }
}
```

#### 4. Presentation Layer (Primary Adapters - Driving)

```csharp
// WebApi/Adapters/Http/OrdersController.cs
namespace MyApp.WebApi.Adapters.Http
{
    /// <summary>
    /// Primary Adapter (HTTP) - drives the application through use cases.
    /// Translates HTTP requests to use case commands.
    /// </summary>
    [ApiController]
    [Route("api/orders")]
    public class OrdersController : ControllerBase
    {
        private readonly ProcessOrderUseCase _processOrderUseCase;
        
        public OrdersController(ProcessOrderUseCase processOrderUseCase)
        {
            _processOrderUseCase = processOrderUseCase;
        }
        
        [HttpPost("{orderId}/process")]
        public async Task<IActionResult> ProcessOrder(Guid orderId)
        {
            // Translate HTTP to command
            var command = new ProcessOrderCommand(new OrderId(orderId));
            
            // Execute use case
            var result = await _processOrderUseCase.ExecuteAsync(command);
            
            // Translate result to HTTP response
            return result.Match(
                onSuccess: _ => Ok(new { Message = "Order processed successfully" }),
                onFailure: error => error switch
                {
                    ProcessOrderError.OrderNotFound => NotFound(new { Error = "Order not found" }),
                    ProcessOrderError.InvalidState e => BadRequest(new { Error = e.Reason }),
                    ProcessOrderError.InsufficientStock => Conflict(new { Error = "Insufficient stock" }),
                    _ => StatusCode(500, new { Error = "Unknown error" })
                });
        }
    }
}

// Console/Adapters/Cli/ProcessOrderCliAdapter.cs
namespace MyApp.Console.Adapters.Cli
{
    /// <summary>
    /// Primary Adapter (CLI) - drives the same application logic through use cases.
    /// Same business logic, different adapter.
    /// </summary>
    public sealed class ProcessOrderCliAdapter
    {
        private readonly ProcessOrderUseCase _processOrderUseCase;
        
        public ProcessOrderCliAdapter(ProcessOrderUseCase processOrderUseCase)
        {
            _processOrderUseCase = processOrderUseCase;
        }
        
        public async Task<int> ExecuteAsync(string[] args)
        {
            if (args.Length == 0 || !Guid.TryParse(args[0], out var orderId))
            {
                System.Console.WriteLine("Usage: process-order <order-id>");
                return 1;
            }
            
            var command = new ProcessOrderCommand(new OrderId(orderId));
            var result = await _processOrderUseCase.ExecuteAsync(command);
            
            return result.Match(
                onSuccess: _ =>
                {
                    System.Console.WriteLine("Order processed successfully");
                    return 0;
                },
                onFailure: error =>
                {
                    System.Console.WriteLine($"Error: {error}");
                    return 1;
                });
        }
    }
}
```

## Swapping Adapters

The power of port-adapter separation is that you can swap implementations without changing the core:

```csharp
// Development: Use in-memory implementations
services.AddScoped<IOrderRepository, InMemoryOrderRepository>();
services.AddScoped<IEmailService, ConsoleEmailAdapter>();  // Logs to console
services.AddScoped<IInventoryService, FakeInventoryAdapter>();  // Always succeeds

// Production: Use real implementations
services.AddScoped<IOrderRepository, SqlOrderRepository>();
services.AddScoped<IEmailService, SendGridEmailAdapter>();
services.AddScoped<IInventoryService, RestInventoryAdapter>();

// Testing: Use test doubles
var fakeOrderRepo = new FakeOrderRepository();
var fakeEmailService = new FakeEmailService();
var fakeInventory = new FakeInventoryService();

var useCase = new ProcessOrderUseCase(
    fakeOrderRepo, 
    fakeEmailService, 
    fakeInventory, 
    new FakeUnitOfWork());
```

## Benefits

1. **Technology Independence**: Swap databases, message queues, APIs without touching business logic
2. **Testability**: Test core logic with fake adapters—no infrastructure needed
3. **Parallel Development**: Teams build adapters while core logic is being defined
4. **Deferrable Decisions**: Delay choosing specific technologies until you have more information
5. **Multiple Interfaces**: Support HTTP, CLI, gRPC using the same core logic

## Why It's a Problem

1. **Tight coupling**: Business logic depends on specific technologies
2. **Hard to test**: Requires real databases, APIs, email servers
3. **Inflexible**: Technology changes require rewriting business logic
4. **Unclear boundaries**: Infrastructure concerns mixed with business rules

## Symptoms

- Domain entities have `[Table]`, `[Column]` attributes
- Business logic has `using` statements for infrastructure libraries
- Can't test without spinning up databases or external services
- Controllers contain complex orchestration logic
- Same workflow can't be invoked from different interfaces (HTTP, CLI, messages)

## See Also

- [Clean Architecture](./clean-architecture.md) — layered architecture pattern
- [Dependency Injection](./dependency-injection.md) — wiring ports to adapters
- [Repository Pattern](./repository-pattern.md) — common secondary port
- [Anti-Corruption Layer](./anti-corruption-layer.md) — protecting domain boundaries
- [Primary vs Secondary Adapters](./primary-secondary-adapters.md) — driving vs driven adapters
- [Hexagonal Ports](./hexagonal-ports.md) — input/output boundaries
