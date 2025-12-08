# Clean Architecture (Onion Architecture / Hexagonal Architecture)

> Large codebases with tangled dependencies make testing and change difficult—organize code in layers where dependencies point inward toward the domain.

## Problem

Applications often couple business logic to infrastructure concerns (databases, frameworks, external APIs). This makes the codebase rigid, fragile, and difficult to test. Business rules become scattered across UI controllers, database queries, and framework code.

## Core Principles

Clean Architecture (also called Onion Architecture or Ports & Adapters) organizes code into concentric layers:

```
┌─────────────────────────────────────────┐
│        Infrastructure Layer             │  ← Outermost
│  (DB, APIs, Frameworks, UI)             │
│  ┌───────────────────────────────────┐  │
│  │    Application Layer              │  │
│  │  (Use Cases, Services)            │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │   Domain Layer              │  │  │  ← Innermost
│  │  │  (Entities, Value Objects,  │  │  │
│  │  │   Business Rules)           │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

Dependencies flow INWARD only →
```

### The Dependency Rule

> Source code dependencies must point only inward, toward higher-level policies.

**This means:**
- Domain layer depends on **nothing**
- Application layer depends only on **Domain**
- Infrastructure depends on **Application** and **Domain**
- Nothing in an inner layer can know anything about an outer layer

## Example

### ❌ Before (Tangled Dependencies)

```csharp
// Business logic directly coupled to Entity Framework
public class OrderController : ControllerBase
{
    private readonly MyDbContext _dbContext;
    
    [HttpPost("orders")]
    public async Task<IActionResult> PlaceOrder(OrderDto dto)
    {
        // Validation in controller
        if (dto.Items.Count == 0)
            return BadRequest("Order must have items");
        
        // Business logic mixed with data access
        var order = new Order
        {
            CustomerId = dto.CustomerId,
            Total = dto.Items.Sum(i => i.Price * i.Quantity),
            Status = "Submitted"
        };
        
        _dbContext.Orders.Add(order);
        
        foreach (var item in dto.Items)
        {
            // More business logic in controller
            var product = await _dbContext.Products.FindAsync(item.ProductId);
            if (product.Stock < item.Quantity)
                return BadRequest("Insufficient stock");
            
            product.Stock -= item.Quantity;
            
            _dbContext.OrderItems.Add(new OrderItem
            {
                OrderId = order.Id,
                ProductId = item.ProductId,
                Quantity = item.Quantity,
                Price = item.Price
            });
        }
        
        await _dbContext.SaveChangesAsync();
        
        // External API call in controller
        await _emailService.SendOrderConfirmationAsync(order.Id);
        
        return Ok(order.Id);
    }
}
```

**Problems:**
- Business logic scattered in controller
- Tightly coupled to Entity Framework
- Hard to test without database
- Can't use domain logic outside HTTP context
- Difficult to switch frameworks or databases

### ✅ After (Clean Architecture)

#### 1. Domain Layer (Innermost - No Dependencies)

```csharp
// Domain/Entities/Order.cs
namespace MyApp.Domain.Entities
{
    /// <summary>
    /// Core business entity with encapsulated business rules.
    /// No dependencies on infrastructure or frameworks.
    /// </summary>
    public sealed class Order
    {
        private readonly List<OrderItem> _items = new();
        
        public OrderId Id { get; }
        public CustomerId CustomerId { get; }
        public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
        public Money Total { get; private set; }
        public OrderStatus Status { get; private set; }
        
        private Order(OrderId id, CustomerId customerId)
        {
            Id = id;
            CustomerId = customerId;
            Total = Money.Zero;
            Status = OrderStatus.Draft;
        }
        
        public static Order Create(CustomerId customerId)
        {
            return new Order(OrderId.New(), customerId);
        }
        
        public Result<Unit, string> AddItem(Product product, int quantity)
        {
            if (Status != OrderStatus.Draft)
                return Result<Unit, string>.Failure("Cannot modify submitted order");
            
            if (quantity <= 0)
                return Result<Unit, string>.Failure("Quantity must be positive");
            
            if (!product.HasSufficientStock(quantity))
                return Result<Unit, string>.Failure("Insufficient stock");
            
            var item = new OrderItem(product.Id, quantity, product.Price);
            _items.Add(item);
            RecalculateTotal();
            
            return Result<Unit, string>.Success(Unit.Value);
        }
        
        public Result<Unit, string> Submit()
        {
            if (Status != OrderStatus.Draft)
                return Result<Unit, string>.Failure("Order already submitted");
            
            if (_items.Count == 0)
                return Result<Unit, string>.Failure("Cannot submit empty order");
            
            Status = OrderStatus.Submitted;
            return Result<Unit, string>.Success(Unit.Value);
        }
        
        private void RecalculateTotal()
        {
            Total = _items
                .Select(i => i.Total)
                .Aggregate(Money.Zero, (sum, price) => sum + price);
        }
    }
}

// Domain/Interfaces/IOrderRepository.cs
namespace MyApp.Domain.Interfaces
{
    /// <summary>
    /// Domain interface - implementation in Infrastructure layer.
    /// Dependency Inversion Principle.
    /// </summary>
    public interface IOrderRepository
    {
        Task<Option<Order>> GetByIdAsync(OrderId id);
        Task SaveAsync(Order order);
    }
    
    public interface IProductRepository
    {
        Task<Option<Product>> GetByIdAsync(ProductId id);
    }
}
```

#### 2. Application Layer (Use Cases)

```csharp
// Application/UseCases/PlaceOrder/PlaceOrderCommand.cs
namespace MyApp.Application.UseCases.PlaceOrder
{
    /// <summary>
    /// Command representing the use case input.
    /// </summary>
    public sealed record PlaceOrderCommand(
        CustomerId CustomerId,
        List<OrderItemDto> Items);
    
    public sealed record OrderItemDto(ProductId ProductId, int Quantity);
}

// Application/UseCases/PlaceOrder/PlaceOrderUseCase.cs
namespace MyApp.Application.UseCases.PlaceOrder
{
    /// <summary>
    /// Use case encapsulates application-specific business rules.
    /// Orchestrates domain objects but contains no business logic itself.
    /// </summary>
    public sealed class PlaceOrderUseCase
    {
        private readonly IOrderRepository _orderRepository;
        private readonly IProductRepository _productRepository;
        private readonly IEmailService _emailService;
        private readonly IUnitOfWork _unitOfWork;
        
        public PlaceOrderUseCase(
            IOrderRepository orderRepository,
            IProductRepository productRepository,
            IEmailService emailService,
            IUnitOfWork unitOfWork)
        {
            _orderRepository = orderRepository;
            _productRepository = productRepository;
            _emailService = emailService;
            _unitOfWork = unitOfWork;
        }
        
        public async Task<Result<OrderId, PlaceOrderError>> ExecuteAsync(
            PlaceOrderCommand command)
        {
            // Create domain entity
            var order = Order.Create(command.CustomerId);
            
            // Orchestrate domain logic
            foreach (var itemDto in command.Items)
            {
                var productResult = await _productRepository.GetByIdAsync(itemDto.ProductId);
                
                var addResult = await productResult.Match(
                    onSome: product =>
                    {
                        var result = order.AddItem(product, itemDto.Quantity);
                        if (result.IsSuccess)
                            product.ReserveStock(itemDto.Quantity);
                        return Task.FromResult(result);
                    },
                    onNone: () => Task.FromResult(
                        Result<Unit, string>.Failure("Product not found")));
                
                if (!addResult.IsSuccess)
                    return Result<OrderId, PlaceOrderError>.Failure(
                        PlaceOrderError.InvalidItem(addResult.Error));
            }
            
            // Submit order (domain logic)
            var submitResult = order.Submit();
            if (!submitResult.IsSuccess)
                return Result<OrderId, PlaceOrderError>.Failure(
                    PlaceOrderError.SubmitFailed(submitResult.Error));
            
            // Persist changes
            await _orderRepository.SaveAsync(order);
            await _unitOfWork.CommitAsync();
            
            // Send notification (side effect)
            await _emailService.SendOrderConfirmationAsync(order.Id, command.CustomerId);
            
            return Result<OrderId, PlaceOrderError>.Success(order.Id);
        }
    }
    
    /// <summary>
    /// Use case specific error type.
    /// </summary>
    public abstract record PlaceOrderError
    {
        public sealed record InvalidItem(string Reason) : PlaceOrderError;
        public sealed record SubmitFailed(string Reason) : PlaceOrderError;
        public sealed record CustomerNotFound : PlaceOrderError;
    }
}

// Application/Interfaces/IEmailService.cs
namespace MyApp.Application.Interfaces
{
    /// <summary>
    /// Application layer interface - implementation in Infrastructure.
    /// </summary>
    public interface IEmailService
    {
        Task SendOrderConfirmationAsync(OrderId orderId, CustomerId customerId);
    }
}
```

#### 3. Infrastructure Layer (Outermost)

```csharp
// Infrastructure/Persistence/OrderRepository.cs
namespace MyApp.Infrastructure.Persistence
{
    /// <summary>
    /// Implementation of domain interface using Entity Framework.
    /// Domain doesn't know about EF - dependency points inward.
    /// </summary>
    public sealed class OrderRepository : IOrderRepository
    {
        private readonly MyDbContext _dbContext;
        
        public async Task<Option<Order>> GetByIdAsync(OrderId id)
        {
            var entity = await _dbContext.Orders
                .Include(o => o.Items)
                .FirstOrDefaultAsync(o => o.Id == id.Value);
            
            if (entity == null)
                return Option<Order>.None;
            
            // Map EF entity to domain entity
            var order = MapToDomain(entity);
            return Option<Order>.Some(order);
        }
        
        public async Task SaveAsync(Order order)
        {
            var entity = MapToEntity(order);
            _dbContext.Orders.Update(entity);
        }
        
        private Order MapToDomain(OrderEntity entity)
        {
            // Translation logic from persistence to domain
            // ...
        }
        
        private OrderEntity MapToEntity(Order order)
        {
            // Translation logic from domain to persistence
            // ...
        }
    }
}

// Infrastructure/Email/SendGridEmailService.cs
namespace MyApp.Infrastructure.Email
{
    /// <summary>
    /// Implementation of application interface using SendGrid.
    /// Application layer doesn't know about SendGrid.
    /// </summary>
    public sealed class SendGridEmailService : IEmailService
    {
        private readonly ISendGridClient _sendGridClient;
        private readonly ICustomerRepository _customerRepository;
        
        public async Task SendOrderConfirmationAsync(OrderId orderId, CustomerId customerId)
        {
            var customer = await _customerRepository.GetByIdAsync(customerId);
            
            await customer.Match(
                onSome: async c =>
                {
                    var message = new SendGridMessage
                    {
                        Subject = "Order Confirmation",
                        PlainTextContent = $"Your order {orderId} has been placed.",
                        To = new List<EmailAddress> { new(c.Email.Value) }
                    };
                    
                    await _sendGridClient.SendEmailAsync(message);
                },
                onNone: () => Task.CompletedTask);
        }
    }
}
```

#### 4. Presentation Layer (Web API)

```csharp
// WebApi/Controllers/OrdersController.cs
namespace MyApp.WebApi.Controllers
{
    /// <summary>
    /// Controller is thin - just translates HTTP to use case commands.
    /// All business logic is in domain and use case layers.
    /// </summary>
    [ApiController]
    [Route("api/orders")]
    public class OrdersController : ControllerBase
    {
        private readonly PlaceOrderUseCase _placeOrderUseCase;
        
        public OrdersController(PlaceOrderUseCase placeOrderUseCase)
        {
            _placeOrderUseCase = placeOrderUseCase;
        }
        
        [HttpPost]
        public async Task<IActionResult> PlaceOrder([FromBody] PlaceOrderRequest request)
        {
            // Translate HTTP request to use case command
            var command = new PlaceOrderCommand(
                new CustomerId(request.CustomerId),
                request.Items.Select(i => new OrderItemDto(
                    new ProductId(i.ProductId),
                    i.Quantity)).ToList());
            
            // Execute use case
            var result = await _placeOrderUseCase.ExecuteAsync(command);
            
            // Translate use case result to HTTP response
            return result.Match(
                onSuccess: orderId => Ok(new { OrderId = orderId.Value }),
                onFailure: error => error switch
                {
                    PlaceOrderError.InvalidItem e => BadRequest(new { Error = e.Reason }),
                    PlaceOrderError.SubmitFailed e => BadRequest(new { Error = e.Reason }),
                    PlaceOrderError.CustomerNotFound => NotFound(new { Error = "Customer not found" }),
                    _ => StatusCode(500, new { Error = "Unknown error" })
                });
        }
    }
    
    // DTOs for HTTP requests (separate from domain)
    public sealed record PlaceOrderRequest(
        Guid CustomerId,
        List<OrderItemRequest> Items);
    
    public sealed record OrderItemRequest(Guid ProductId, int Quantity);
}
```

## Project Structure (Screaming Architecture)

```
Solution: MyApp
├── MyApp.Domain/                          ← Core business logic (no dependencies)
│   ├── Entities/
│   │   ├── Order.cs
│   │   ├── OrderItem.cs
│   │   ├── Customer.cs
│   │   └── Product.cs
│   ├── ValueObjects/
│   │   ├── OrderId.cs
│   │   ├── Money.cs
│   │   └── Email.cs
│   ├── Interfaces/                        ← Abstractions (DIP)
│   │   ├── IOrderRepository.cs
│   │   └── IProductRepository.cs
│   └── Events/
│       └── OrderPlacedEvent.cs
│
├── MyApp.Application/                     ← Use cases (depends on Domain only)
│   ├── UseCases/
│   │   ├── PlaceOrder/
│   │   │   ├── PlaceOrderCommand.cs
│   │   │   ├── PlaceOrderUseCase.cs
│   │   │   └── PlaceOrderError.cs
│   │   ├── CancelOrder/
│   │   │   ├── CancelOrderCommand.cs
│   │   │   └── CancelOrderUseCase.cs
│   │   └── GetOrderDetails/
│   │       ├── GetOrderDetailsQuery.cs
│   │       └── GetOrderDetailsQueryHandler.cs
│   ├── Interfaces/                        ← Application abstractions
│   │   ├── IEmailService.cs
│   │   └── IUnitOfWork.cs
│   └── Common/
│       ├── Result.cs
│       └── Option.cs
│
├── MyApp.Infrastructure/                  ← External concerns (depends on Application & Domain)
│   ├── Persistence/
│   │   ├── MyDbContext.cs
│   │   ├── OrderRepository.cs
│   │   ├── ProductRepository.cs
│   │   └── UnitOfWork.cs
│   ├── Email/
│   │   └── SendGridEmailService.cs
│   ├── ExternalApis/
│   │   └── PaymentGatewayAdapter.cs
│   └── DependencyInjection.cs
│
└── MyApp.WebApi/                          ← HTTP interface (depends on Application)
    ├── Controllers/
    │   └── OrdersController.cs
    ├── Middleware/
    │   └── ErrorHandlingMiddleware.cs
    ├── DTOs/
    │   ├── PlaceOrderRequest.cs
    │   └── OrderResponse.cs
    ├── Program.cs
    └── appsettings.json
```

## Dependency Injection Configuration

```csharp
// Infrastructure/DependencyInjection.cs
namespace MyApp.Infrastructure
{
    public static class DependencyInjection
    {
        public static IServiceCollection AddInfrastructure(
            this IServiceCollection services,
            IConfiguration configuration)
        {
            // Database
            services.AddDbContext<MyDbContext>(options =>
                options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));
            
            // Repositories (Domain interfaces → Infrastructure implementations)
            services.AddScoped<IOrderRepository, OrderRepository>();
            services.AddScoped<IProductRepository, ProductRepository>();
            services.AddScoped<IUnitOfWork, UnitOfWork>();
            
            // External services (Application interfaces → Infrastructure implementations)
            services.AddScoped<IEmailService, SendGridEmailService>();
            
            return services;
        }
    }
}

// Application/DependencyInjection.cs
namespace MyApp.Application
{
    public static class DependencyInjection
    {
        public static IServiceCollection AddApplication(this IServiceCollection services)
        {
            // Register use cases
            services.AddScoped<PlaceOrderUseCase>();
            services.AddScoped<CancelOrderUseCase>();
            services.AddScoped<GetOrderDetailsQueryHandler>();
            
            return services;
        }
    }
}

// WebApi/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddApplication();           // Application layer services
builder.Services.AddInfrastructure(builder.Configuration);  // Infrastructure implementations

var app = builder.Build();
app.Run();
```

## Benefits of Clean Architecture

### 1. Testability

```csharp
// Test domain logic without any infrastructure
[Fact]
public void Order_AddItem_WithInsufficientStock_ReturnsError()
{
    // No database, no HTTP, no frameworks needed
    var order = Order.Create(new CustomerId(Guid.NewGuid()));
    var product = Product.Create("Widget", Money.USD(10), stock: 5);
    
    var result = order.AddItem(product, quantity: 10);
    
    Assert.False(result.IsSuccess);
    Assert.Equal("Insufficient stock", result.Error);
}

// Test use case with fake repositories
[Fact]
public async Task PlaceOrder_WithValidItems_CreatesOrder()
{
    var fakeOrderRepo = new FakeOrderRepository();
    var fakeProductRepo = new FakeProductRepository();
    fakeProductRepo.AddProduct(Product.Create("Widget", Money.USD(10), stock: 100));
    
    var useCase = new PlaceOrderUseCase(
        fakeOrderRepo,
        fakeProductRepo,
        new FakeEmailService(),
        new FakeUnitOfWork());
    
    var command = new PlaceOrderCommand(
        new CustomerId(Guid.NewGuid()),
        new List<OrderItemDto> { new(ProductId.New(), 5) });
    
    var result = await useCase.ExecuteAsync(command);
    
    Assert.True(result.IsSuccess);
    Assert.Single(fakeOrderRepo.SavedOrders);
}
```

### 2. Framework Independence

```csharp
// Easy to switch from Entity Framework to Dapper or MongoDB
// Just create new implementation of IOrderRepository
public class DapperOrderRepository : IOrderRepository
{
    public async Task<Option<Order>> GetByIdAsync(OrderId id)
    {
        // Different persistence mechanism, same interface
        using var connection = new SqlConnection(_connectionString);
        var sql = "SELECT * FROM Orders WHERE Id = @Id";
        var entity = await connection.QueryFirstOrDefaultAsync<OrderEntity>(sql, new { Id = id.Value });
        // ...
    }
}

// Register in DI—domain and application layers unchanged
services.AddScoped<IOrderRepository, DapperOrderRepository>();
```

### 3. Business Logic Centralization

All business rules in one place (Domain layer), easy to find and modify:
- Order validation rules: `Order.AddItem()`
- Status transitions: `Order.Submit()`, `Order.Ship()`
- Invariants: `RecalculateTotal()` is private

### 4. Parallel Development

- UI team works on controllers/views
- Domain team works on business logic
- Infrastructure team works on database/APIs
- All without stepping on each other's toes

## Why It's a Problem

1. **Tangled dependencies**: Business logic mixed with infrastructure
2. **Hard to test**: Requires database, HTTP context, external APIs
3. **Framework lock-in**: Can't switch technologies without rewriting
4. **Scattered logic**: Business rules duplicated across layers
5. **Rigid design**: Changes ripple through entire codebase

## Symptoms

- Controllers with hundreds of lines of business logic
- Entity Framework models used as domain models
- Difficulty testing without spinning up database
- `using` statements for infrastructure in domain classes
- Business rules duplicated in multiple controllers

## Benefits

- **Testable**: Test business logic without infrastructure
- **Flexible**: Swap frameworks, databases, external services easily
- **Maintainable**: Business rules centralized and explicit
- **Clear boundaries**: Each layer has well-defined responsibility
- **Independent development**: Teams work on layers independently

## Common Pitfalls

### 1. Anemic Domain Model

```csharp
// ❌ Domain entity with no behavior (just data)
public class Order
{
    public Guid Id { get; set; }
    public List<OrderItem> Items { get; set; }
    public decimal Total { get; set; }
}

// Business logic in service instead of domain
public class OrderService
{
    public void AddItem(Order order, OrderItem item)
    {
        order.Items.Add(item);
        order.Total = order.Items.Sum(i => i.Price * i.Quantity);
    }
}

// ✅ Rich domain model with behavior
public class Order
{
    public Result<Unit, string> AddItem(Product product, int quantity)
    {
        // Business logic here, in the domain
    }
}
```

### 2. Leaking Persistence Concerns

```csharp
// ❌ Domain entity knows about database
public class Order
{
    [Key]
    [Column("order_id")]
    public Guid Id { get; set; }  // EF attributes in domain
}

// ✅ Separate persistence model from domain
namespace Infrastructure.Persistence
{
    public class OrderEntity
    {
        [Key]
        [Column("order_id")]
        public Guid Id { get; set; }
    }
}
```

### 3. Application Logic in Controllers

```csharp
// ❌ Use case logic in controller
[HttpPost]
public async Task<IActionResult> PlaceOrder(PlaceOrderRequest request)
{
    var order = Order.Create(request.CustomerId);
    foreach (var item in request.Items)
    {
        // Orchestration logic in controller
        var product = await _productRepo.GetByIdAsync(item.ProductId);
        order.AddItem(product, item.Quantity);
    }
    await _orderRepo.SaveAsync(order);
    return Ok(order.Id);
}

// ✅ Thin controller delegates to use case
[HttpPost]
public async Task<IActionResult> PlaceOrder(PlaceOrderRequest request)
{
    var command = MapToCommand(request);
    var result = await _placeOrderUseCase.ExecuteAsync(command);
    return MapToResponse(result);
}
```

## See Also

- [Dependency Injection](./dependency-injection.md) — implementing the dependency rule
- [Repository Pattern](./repository-pattern.md) — data access abstraction
- [Anti-Corruption Layer](./anti-corruption-layer.md) — protecting domain boundaries
- [DTO vs Domain Boundary](./dto-domain-boundary.md) — separating API from domain
- [Bounded Contexts](./bounded-contexts.md) — strategic domain boundaries
- [Aggregate Roots](./aggregate-roots.md) — transactional consistency boundaries
- [Domain Events](./domain-events.md) — decoupling side effects
- [Use Cases / Application Services](./use-cases.md) — orchestrating domain logic
