# Application Service Layer Isolation

> Application services orchestrate domain logic without containing business rules themselves—they coordinate use cases, manage transactions, and translate between external concerns and the domain.

## Problem

When application-level concerns (orchestration, transaction management, authorization checks) are mixed with domain logic or pushed into controllers, the code becomes hard to test and reuse. The application layer should be a thin orchestration layer that coordinates domain objects and infrastructure services without implementing business rules.

## Core Concept

The Application Service Layer sits between the presentation layer and the domain layer:

```
┌──────────────────────────────┐
│   Presentation Layer         │  ← Controllers, CLI, gRPC
│   (HTTP, CLI, Messages)      │
└────────────┬─────────────────┘
             │
┌────────────▼─────────────────┐
│  Application Service Layer   │  ← Use Cases, Commands, Queries
│  - Orchestration             │
│  - Transaction Management    │
│  - Authorization Checks      │
│  - Input/Output Translation  │
└────────────┬─────────────────┘
             │
┌────────────▼─────────────────┐
│      Domain Layer            │  ← Business Logic, Entities
│  - Business Rules            │
│  - Domain Logic              │
│  - Invariant Enforcement     │
└──────────────────────────────┘
```

**Application Services:**
- Contain NO business logic
- Orchestrate domain objects
- Manage transactions (UnitOfWork)
- Perform authorization checks
- Translate between DTOs and domain models
- Coordinate infrastructure services

**Domain:**
- Contains ALL business logic
- No knowledge of transactions, persistence, or external services
- Pure business rules and invariants

## Example

### ❌ Before (Mixed Concerns)

```csharp
// Domain logic mixed into application service
public class OrderApplicationService
{
    private readonly IOrderRepository _orderRepository;
    private readonly IDbContext _dbContext;
    
    public async Task<OrderDto> CreateOrder(CreateOrderDto dto)
    {
        // ❌ Business logic in application service
        if (dto.Items.Count == 0)
            throw new InvalidOperationException("Order must have items");
        
        if (dto.Items.Sum(i => i.Quantity) > 100)
            throw new InvalidOperationException("Cannot order more than 100 items");
        
        // ❌ Transaction management mixed with business logic
        using var transaction = await _dbContext.BeginTransactionAsync();
        
        try
        {
            var order = new Order
            {
                Id = Guid.NewGuid(),
                CustomerId = dto.CustomerId,
                // ❌ Calculation in application service (should be in domain)
                Total = dto.Items.Sum(i => i.Price * i.Quantity),
                Items = dto.Items.Select(i => new OrderItem
                {
                    ProductId = i.ProductId,
                    Quantity = i.Quantity,
                    Price = i.Price
                }).ToList()
            };
            
            await _orderRepository.AddAsync(order);
            await transaction.CommitAsync();
            
            return new OrderDto
            {
                Id = order.Id,
                Total = order.Total
            };
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

**Problems:**
- Business rules (validation, calculations) in application service instead of domain
- Transaction management scattered throughout
- Hard to test business logic separately
- Can't reuse domain logic outside this specific workflow
- Application service is doing too much

### ✅ After (Application Service Layer Isolation)

#### 1. Domain Layer (Pure Business Logic)

```csharp
// Domain/Entities/Order.cs
namespace MyApp.Domain.Entities
{
    /// <summary>
    /// Domain entity with ALL business logic encapsulated.
    /// Application service cannot bypass these rules.
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
        
        public static Result<Order, string> Create(CustomerId customerId)
        {
            // Domain validation
            return Result<Order, string>.Success(
                new Order(OrderId.New(), customerId));
        }
        
        public Result<Unit, string> AddItem(
            ProductId productId, 
            int quantity, 
            Money price)
        {
            // ✅ Business rule in domain
            if (quantity <= 0)
                return Result<Unit, string>.Failure(
                    "Quantity must be positive");
            
            // ✅ Business rule in domain
            if (_items.Sum(i => i.Quantity) + quantity > 100)
                return Result<Unit, string>.Failure(
                    "Cannot order more than 100 items total");
            
            var item = new OrderItem(productId, quantity, price);
            _items.Add(item);
            
            // ✅ Calculation in domain
            RecalculateTotal();
            
            return Result<Unit, string>.Success(Unit.Value);
        }
        
        public Result<Unit, string> Submit()
        {
            // ✅ Business rule in domain
            if (_items.Count == 0)
                return Result<Unit, string>.Failure(
                    "Cannot submit empty order");
            
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
```

#### 2. Application Layer (Pure Orchestration)

```csharp
// Application/UseCases/CreateOrder/CreateOrderCommand.cs
namespace MyApp.Application.UseCases.CreateOrder
{
    /// <summary>
    /// Input to the application service.
    /// Pure data - no behavior.
    /// </summary>
    public sealed record CreateOrderCommand(
        CustomerId CustomerId,
        List<OrderItemCommand> Items);
    
    public sealed record OrderItemCommand(
        ProductId ProductId,
        int Quantity);
}

// Application/UseCases/CreateOrder/CreateOrderUseCase.cs
namespace MyApp.Application.UseCases.CreateOrder
{
    /// <summary>
    /// Application service that orchestrates domain logic.
    /// Contains NO business rules - only coordination.
    /// </summary>
    public sealed class CreateOrderUseCase
    {
        private readonly IOrderRepository _orderRepository;
        private readonly IProductRepository _productRepository;
        private readonly IUnitOfWork _unitOfWork;
        private readonly IAuthorizationService _authorizationService;
        
        public CreateOrderUseCase(
            IOrderRepository orderRepository,
            IProductRepository productRepository,
            IUnitOfWork unitOfWork,
            IAuthorizationService authorizationService)
        {
            _orderRepository = orderRepository;
            _productRepository = productRepository;
            _unitOfWork = unitOfWork;
            _authorizationService = authorizationService;
        }
        
        public async Task<Result<OrderId, CreateOrderError>> ExecuteAsync(
            CreateOrderCommand command,
            UserId userId)
        {
            // ✅ Application concern: authorization check
            if (!await _authorizationService.CanCreateOrderAsync(userId, command.CustomerId))
                return Result<OrderId, CreateOrderError>.Failure(
                    new CreateOrderError.Unauthorized());
            
            // ✅ Application concern: orchestration starts
            // ✅ Domain logic: create order (business rules in domain)
            var orderResult = Order.Create(command.CustomerId);
            if (!orderResult.IsSuccess)
                return Result<OrderId, CreateOrderError>.Failure(
                    new CreateOrderError.InvalidOrder(orderResult.Error));
            
            var order = orderResult.Value;
            
            // ✅ Application concern: orchestrate adding items
            foreach (var itemCommand in command.Items)
            {
                // ✅ Application concern: fetch product
                var productResult = await _productRepository.GetByIdAsync(
                    itemCommand.ProductId);
                
                var product = await productResult.Match(
                    onSome: p => Task.FromResult<Product?>(p),
                    onNone: () => Task.FromResult<Product?>(null));
                
                if (product == null)
                    return Result<OrderId, CreateOrderError>.Failure(
                        new CreateOrderError.ProductNotFound(itemCommand.ProductId));
                
                // ✅ Domain logic: add item (business rules in domain)
                var addResult = order.AddItem(
                    itemCommand.ProductId,
                    itemCommand.Quantity,
                    product.Price);
                
                if (!addResult.IsSuccess)
                    return Result<OrderId, CreateOrderError>.Failure(
                        new CreateOrderError.InvalidItem(addResult.Error));
            }
            
            // ✅ Domain logic: submit order (business rule in domain)
            var submitResult = order.Submit();
            if (!submitResult.IsSuccess)
                return Result<OrderId, CreateOrderError>.Failure(
                    new CreateOrderError.SubmitFailed(submitResult.Error));
            
            // ✅ Application concern: transaction management
            await _orderRepository.SaveAsync(order);
            await _unitOfWork.CommitAsync();
            
            return Result<OrderId, CreateOrderError>.Success(order.Id);
        }
    }
    
    /// <summary>
    /// Application-level errors.
    /// Different from domain errors - these include infrastructure concerns.
    /// </summary>
    public abstract record CreateOrderError
    {
        public sealed record Unauthorized : CreateOrderError;
        public sealed record ProductNotFound(ProductId ProductId) : CreateOrderError;
        public sealed record InvalidOrder(string Reason) : CreateOrderError;
        public sealed record InvalidItem(string Reason) : CreateOrderError;
        public sealed record SubmitFailed(string Reason) : CreateOrderError;
    }
}

// Application/Services/IAuthorizationService.cs
namespace MyApp.Application.Services
{
    /// <summary>
    /// Application-level service for authorization.
    /// Not a domain concern - belongs in application layer.
    /// </summary>
    public interface IAuthorizationService
    {
        Task<bool> CanCreateOrderAsync(UserId userId, CustomerId customerId);
    }
}

// Application/Services/IUnitOfWork.cs
namespace MyApp.Application.Services
{
    /// <summary>
    /// Application-level transaction management.
    /// Domain doesn't know about transactions.
    /// </summary>
    public interface IUnitOfWork
    {
        Task CommitAsync();
        Task RollbackAsync();
    }
}
```

#### 3. Presentation Layer (Thin Controllers)

```csharp
// WebApi/Controllers/OrdersController.cs
namespace MyApp.WebApi.Controllers
{
    [ApiController]
    [Route("api/orders")]
    public class OrdersController : ControllerBase
    {
        private readonly CreateOrderUseCase _createOrderUseCase;
        private readonly ICurrentUserService _currentUserService;
        
        public OrdersController(
            CreateOrderUseCase createOrderUseCase,
            ICurrentUserService currentUserService)
        {
            _createOrderUseCase = createOrderUseCase;
            _currentUserService = currentUserService;
        }
        
        [HttpPost]
        public async Task<IActionResult> CreateOrder(
            [FromBody] CreateOrderRequest request)
        {
            // ✅ Controller: translate HTTP to command
            var command = new CreateOrderCommand(
                new CustomerId(request.CustomerId),
                request.Items.Select(i => new OrderItemCommand(
                    new ProductId(i.ProductId),
                    i.Quantity)).ToList());
            
            // ✅ Controller: get current user context
            var userId = _currentUserService.GetCurrentUserId();
            
            // ✅ Delegate to application service
            var result = await _createOrderUseCase.ExecuteAsync(command, userId);
            
            // ✅ Controller: translate result to HTTP response
            return result.Match(
                onSuccess: orderId => CreatedAtAction(
                    nameof(GetOrder),
                    new { id = orderId.Value },
                    new { OrderId = orderId.Value }),
                onFailure: error => error switch
                {
                    CreateOrderError.Unauthorized => 
                        Forbid(),
                    CreateOrderError.ProductNotFound e => 
                        NotFound(new { Error = $"Product {e.ProductId} not found" }),
                    CreateOrderError.InvalidOrder e => 
                        BadRequest(new { Error = e.Reason }),
                    CreateOrderError.InvalidItem e => 
                        BadRequest(new { Error = e.Reason }),
                    CreateOrderError.SubmitFailed e => 
                        BadRequest(new { Error = e.Reason }),
                    _ => 
                        StatusCode(500, new { Error = "Unknown error" })
                });
        }
        
        [HttpGet("{id}")]
        public async Task<IActionResult> GetOrder(Guid id)
        {
            // Implementation for route reference
            return Ok();
        }
    }
    
    // DTO for HTTP request
    public sealed record CreateOrderRequest(
        Guid CustomerId,
        List<OrderItemRequest> Items);
    
    public sealed record OrderItemRequest(Guid ProductId, int Quantity);
}
```

## Application Service Responsibilities

| Responsibility | Application Service | Domain |
|----------------|---------------------|--------|
| **Business Rules** | ❌ Never | ✅ Always |
| **Orchestration** | ✅ Yes | ❌ No |
| **Transaction Management** | ✅ Yes (UnitOfWork) | ❌ No |
| **Authorization** | ✅ Yes | ❌ No |
| **Fetch Dependencies** | ✅ Yes (Repositories) | ❌ No |
| **DTO Translation** | ✅ Yes | ❌ No |
| **Logging** | ✅ Yes | ❌ No |
| **Validation** | Input validation | Business invariants |

## Benefits

1. **Testability**: Domain logic testable in isolation without infrastructure
2. **Reusability**: Same domain logic works from HTTP, CLI, messages
3. **Clear Boundaries**: Business rules can't leak into application concerns
4. **Transaction Control**: Centralized transaction management in application layer
5. **Authorization Separation**: Security checks separate from business logic

## Why It's a Problem

1. **Mixed concerns**: Business logic scattered across application and domain
2. **Hard to test**: Can't test business rules without infrastructure
3. **Duplication**: Business rules duplicated across different workflows
4. **Unclear boundaries**: No clear place for orchestration vs business logic

## Symptoms

- Application services contain if-else chains with business rules
- Domain entities are anemic (just getters/setters)
- Transaction management mixed with business logic
- Same validation appears in multiple places
- Can't test business logic without database

## See Also

- [Clean Architecture](./clean-architecture.md) — layered architecture pattern
- [Use Cases](./use-cases.md) — application services pattern
- [Rich Domain Models](./rich-domain-models.md) — behavior in domain entities
- [Domain Services](./domain-services.md) — vs application services
- [Repository Pattern](./repository-pattern.md) — fetching aggregates
