# Domain Model Isolation (Pure Domain Logic Without Infrastructure Concerns)

> Domain logic mixed with database, HTTP, or framework code makes testing hard and couples business rules to infrastructure—isolate pure domain models from all external concerns.

## Problem

Domain logic is tangled with infrastructure concerns like database access, HTTP clients, logging, or framework-specific code. This makes business rules hard to test, understand, and change independently of infrastructure choices.

## Example

### ❌ Before

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;
using System.Net.Http;

// Domain entity polluted with infrastructure concerns
public class Order : DbContext  // Inheriting from EF!
{
    private readonly ILogger<Order> _logger;
    private readonly HttpClient _httpClient;
    
    public Order(ILogger<Order> logger, HttpClient httpClient)
    {
        _logger = logger;
        _httpClient = httpClient;
    }
    
    public int Id { get; set; }
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    
    [NotMapped]  // EF attribute in domain model!
    public decimal Total => Items.Sum(i => i.Price * i.Quantity);
    
    public async Task<bool> PlaceOrder()
    {
        // Domain logic mixed with infrastructure
        _logger.LogInformation($"Placing order {Id}");
        
        // HTTP call in domain logic!
        var response = await _httpClient.PostAsJsonAsync(
            "https://api.payment.com/charge",
            new { OrderId = Id, Amount = Total });
        
        if (!response.IsSuccessStatusCode)
        {
            _logger.LogError("Payment failed");
            return false;
        }
        
        // Database save in domain logic!
        await SaveChangesAsync();
        
        return true;
    }
}
```

**Problems:**
- Domain entity depends on EF, logging, and HTTP
- Can't test business logic without mocking infrastructure
- Business rules hidden behind technical noise
- Changing database or HTTP library requires changing domain
- Impossible to unit test without extensive mocking

### ✅ After

```csharp
// ============================================
// DOMAIN LAYER (Pure, no infrastructure dependencies)
// ============================================
namespace Orders.Domain
{
    /// <summary>
    /// Pure domain entity with only business logic.
    /// No infrastructure dependencies.
    /// </summary>
    public sealed class Order
    {
        private readonly List<OrderLine> _lines = new();
        
        public OrderId Id { get; }
        public CustomerId CustomerId { get; }
        public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
        public OrderStatus Status { get; private set; }
        
        // Pure calculation, no infrastructure
        public Money Total => _lines
            .Select(line => line.LineTotal)
            .Aggregate(Money.Zero, (sum, total) => sum + total);
        
        private Order(OrderId id, CustomerId customerId)
        {
            Id = id;
            CustomerId = customerId;
            Status = OrderStatus.Draft;
        }
        
        public static Order Create(OrderId id, CustomerId customerId) =>
            new Order(id, customerId);
        
        // Pure business logic
        public Result<Unit, OrderError> AddLine(
            ProductId productId,
            int quantity,
            Money unitPrice)
        {
            if (Status != OrderStatus.Draft)
                return Result<Unit, OrderError>.Failure(
                    OrderError.CannotModifySubmittedOrder);
            
            if (quantity <= 0)
                return Result<Unit, OrderError>.Failure(
                    OrderError.InvalidQuantity);
            
            var line = new OrderLine(productId, quantity, unitPrice);
            _lines.Add(line);
            
            return Result<Unit, OrderError>.Success(Unit.Value);
        }
        
        // Pure business logic
        public Result<Unit, OrderError> Submit()
        {
            if (Status != OrderStatus.Draft)
                return Result<Unit, OrderError>.Failure(
                    OrderError.OrderAlreadySubmitted);
            
            if (_lines.Count == 0)
                return Result<Unit, OrderError>.Failure(
                    OrderError.EmptyOrder);
            
            if (Total.Amount < 10.00m)
                return Result<Unit, OrderError>.Failure(
                    OrderError.BelowMinimumAmount);
            
            Status = OrderStatus.Submitted;
            return Result<Unit, OrderError>.Success(Unit.Value);
        }
        
        // Pure business logic
        public Result<Unit, OrderError> Confirm()
        {
            if (Status != OrderStatus.Submitted)
                return Result<Unit, OrderError>.Failure(
                    OrderError.CannotConfirmUnsubmittedOrder);
            
            Status = OrderStatus.Confirmed;
            return Result<Unit, OrderError>.Success(Unit.Value);
        }
    }
    
    public sealed record OrderLine(
        ProductId ProductId,
        int Quantity,
        Money UnitPrice)
    {
        public Money LineTotal => UnitPrice * Quantity;
    }
    
    // Domain errors as types
    public enum OrderError
    {
        CannotModifySubmittedOrder,
        InvalidQuantity,
        OrderAlreadySubmitted,
        EmptyOrder,
        BelowMinimumAmount,
        CannotConfirmUnsubmittedOrder
    }
    
    // Domain interfaces (not infrastructure implementations)
    public interface IOrderRepository
    {
        Task<Option<Order>> GetByIdAsync(OrderId id);
        Task SaveAsync(Order order);
    }
    
    public interface IPaymentGateway
    {
        Task<Result<PaymentConfirmation, string>> ProcessPayment(
            OrderId orderId,
            Money amount);
    }
}

// ============================================
// APPLICATION LAYER (Orchestration, no business logic)
// ============================================
namespace Orders.Application
{
    using Orders.Domain;
    
    /// <summary>
    /// Application service: Orchestrates domain and infrastructure.
    /// Contains workflow logic, not business rules.
    /// </summary>
    public sealed class PlaceOrderHandler
    {
        private readonly IOrderRepository _orderRepo;
        private readonly IPaymentGateway _paymentGateway;
        private readonly IEventPublisher _eventPublisher;
        
        public PlaceOrderHandler(
            IOrderRepository orderRepo,
            IPaymentGateway paymentGateway,
            IEventPublisher eventPublisher)
        {
            _orderRepo = orderRepo;
            _paymentGateway = paymentGateway;
            _eventPublisher = eventPublisher;
        }
        
        public async Task<Result<OrderId, string>> Handle(PlaceOrderCommand cmd)
        {
            // Load aggregate
            var orderOpt = await _orderRepo.GetByIdAsync(cmd.OrderId);
            if (orderOpt.IsNone)
                return Result<OrderId, string>.Failure("Order not found");
            
            var order = orderOpt.Value;
            
            // Execute pure business logic
            var submitResult = order.Submit();
            if (!submitResult.IsSuccess)
                return Result<OrderId, string>.Failure(
                    submitResult.Error.ToString());
            
            // Infrastructure: Process payment
            var paymentResult = await _paymentGateway.ProcessPayment(
                order.Id,
                order.Total);
            
            if (!paymentResult.IsSuccess)
                return Result<OrderId, string>.Failure(paymentResult.Error);
            
            // Execute pure business logic
            var confirmResult = order.Confirm();
            if (!confirmResult.IsSuccess)
                return Result<OrderId, string>.Failure(
                    confirmResult.Error.ToString());
            
            // Infrastructure: Save
            await _orderRepo.SaveAsync(order);
            
            // Infrastructure: Publish event
            await _eventPublisher.PublishAsync(new OrderPlacedEvent(
                order.Id,
                order.CustomerId,
                order.Total));
            
            return Result<OrderId, string>.Success(order.Id);
        }
    }
}

// ============================================
// INFRASTRUCTURE LAYER (Implementation details)
// ============================================
namespace Orders.Infrastructure
{
    using Microsoft.EntityFrameworkCore;
    using Microsoft.Extensions.Logging;
    using Orders.Domain;
    
    /// <summary>
    /// EF implementation hidden in infrastructure layer.
    /// Domain doesn't know about EF.
    /// </summary>
    public class OrderRepository : IOrderRepository
    {
        private readonly OrderDbContext _context;
        private readonly ILogger<OrderRepository> _logger;
        
        public OrderRepository(
            OrderDbContext context,
            ILogger<OrderRepository> logger)
        {
            _context = context;
            _logger = logger;
        }
        
        public async Task<Option<Order>> GetByIdAsync(OrderId id)
        {
            _logger.LogDebug($"Loading order {id}");
            
            // Map from EF entity to domain model
            var entity = await _context.Orders
                .Include(o => o.Lines)
                .FirstOrDefaultAsync(o => o.Id == id.Value);
            
            if (entity == null)
                return Option<Order>.None();
            
            return Option<Order>.Some(entity.ToDomainModel());
        }
        
        public async Task SaveAsync(Order order)
        {
            _logger.LogDebug($"Saving order {order.Id}");
            
            // Map from domain model to EF entity
            var entity = OrderEntity.FromDomainModel(order);
            
            _context.Orders.Update(entity);
            await _context.SaveChangesAsync();
        }
    }
    
    /// <summary>
    /// EF entity for persistence (not the domain model).
    /// </summary>
    internal class OrderEntity
    {
        public Guid Id { get; set; }
        public Guid CustomerId { get; set; }
        public string Status { get; set; }
        public List<OrderLineEntity> Lines { get; set; }
        
        public Order ToDomainModel()
        {
            // Convert EF entity to domain model
            var order = Order.Create(
                new OrderId(Id),
                new CustomerId(CustomerId));
            
            foreach (var line in Lines)
            {
                order.AddLine(
                    new ProductId(line.ProductId),
                    line.Quantity,
                    Money.USD(line.UnitPrice));
            }
            
            return order;
        }
        
        public static OrderEntity FromDomainModel(Order order)
        {
            return new OrderEntity
            {
                Id = order.Id.Value,
                CustomerId = order.CustomerId.Value,
                Status = order.Status.ToString(),
                Lines = order.Lines
                    .Select(l => new OrderLineEntity
                    {
                        ProductId = l.ProductId.Value,
                        Quantity = l.Quantity,
                        UnitPrice = l.UnitPrice.Amount
                    })
                    .ToList()
            };
        }
    }
    
    internal class OrderLineEntity
    {
        public Guid ProductId { get; set; }
        public int Quantity { get; set; }
        public decimal UnitPrice { get; set; }
    }
    
    /// <summary>
    /// HTTP client implementation hidden in infrastructure.
    /// Domain doesn't know about HTTP.
    /// </summary>
    public class HttpPaymentGateway : IPaymentGateway
    {
        private readonly HttpClient _httpClient;
        private readonly ILogger<HttpPaymentGateway> _logger;
        
        public HttpPaymentGateway(
            HttpClient httpClient,
            ILogger<HttpPaymentGateway> logger)
        {
            _httpClient = httpClient;
            _logger = logger;
        }
        
        public async Task<Result<PaymentConfirmation, string>> ProcessPayment(
            OrderId orderId,
            Money amount)
        {
            _logger.LogInformation($"Processing payment for order {orderId}");
            
            try
            {
                var response = await _httpClient.PostAsJsonAsync(
                    "https://api.payment.com/charge",
                    new
                    {
                        OrderId = orderId.Value,
                        Amount = amount.Amount,
                        Currency = amount.Currency.Code
                    });
                
                if (!response.IsSuccessStatusCode)
                {
                    _logger.LogError("Payment failed");
                    return Result<PaymentConfirmation, string>.Failure(
                        "Payment processing failed");
                }
                
                var confirmation = await response.Content
                    .ReadFromJsonAsync<PaymentConfirmationDto>();
                
                return Result<PaymentConfirmation, string>.Success(
                    new PaymentConfirmation(confirmation.TransactionId));
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Payment error");
                return Result<PaymentConfirmation, string>.Failure(ex.Message);
            }
        }
    }
}
```

## Testing Pure Domain Logic

```csharp
// ✅ Unit test: Pure domain logic, no mocking required
public class OrderTests
{
    [Fact]
    public void AddLine_ToDraftOrder_Succeeds()
    {
        // Arrange: Pure object creation, no dependencies
        var order = Order.Create(OrderId.New(), CustomerId.New());
        
        // Act: Pure business logic
        var result = order.AddLine(
            ProductId.New(),
            5,
            Money.USD(10.00m));
        
        // Assert: Simple verification
        Assert.True(result.IsSuccess);
        Assert.Equal(1, order.Lines.Count);
        Assert.Equal(Money.USD(50.00m), order.Total);
    }
    
    [Fact]
    public void Submit_EmptyOrder_ReturnsError()
    {
        // Arrange
        var order = Order.Create(OrderId.New(), CustomerId.New());
        
        // Act
        var result = order.Submit();
        
        // Assert
        Assert.False(result.IsSuccess);
        Assert.Equal(OrderError.EmptyOrder, result.Error);
    }
    
    [Fact]
    public void AddLine_ToSubmittedOrder_ReturnsError()
    {
        // Arrange
        var order = Order.Create(OrderId.New(), CustomerId.New());
        order.AddLine(ProductId.New(), 5, Money.USD(10.00m));
        order.Submit();
        
        // Act
        var result = order.AddLine(ProductId.New(), 1, Money.USD(5.00m));
        
        // Assert
        Assert.False(result.IsSuccess);
        Assert.Equal(OrderError.CannotModifySubmittedOrder, result.Error);
    }
}

// ❌ Before: Infrastructure test masquerading as unit test
public class OrderTestsWithInfrastructure
{
    [Fact]
    public async Task PlaceOrder_WithPayment_Succeeds()
    {
        // Massive setup required
        var logger = new Mock<ILogger<Order>>();
        var httpClient = new Mock<HttpClient>();
        var dbContext = new Mock<DbContext>();
        
        // Configure complex mocks
        httpClient.Setup(c => c.PostAsJsonAsync(
            It.IsAny<string>(),
            It.IsAny<object>()))
            .ReturnsAsync(new HttpResponseMessage(HttpStatusCode.OK));
        
        // Test business logic tangled with infrastructure
        var order = new Order(logger.Object, httpClient.Object);
        var result = await order.PlaceOrder();
        
        Assert.True(result);
    }
}
```

## Dependency Direction Rules

```csharp
// ✅ GOOD: Dependencies point inward
// Infrastructure → Application → Domain

// Domain Layer: No dependencies
namespace Domain
{
    public interface IOrderRepository { }  // Interface defined in domain
    public class Order { }  // Pure domain logic
}

// Application Layer: Depends on Domain only
namespace Application
{
    using Domain;  // ✓ OK
    
    public class PlaceOrderHandler
    {
        private readonly IOrderRepository _repo;  // Domain interface
    }
}

// Infrastructure Layer: Depends on Domain and Application
namespace Infrastructure
{
    using Domain;  // ✓ OK
    using Application;  // ✓ OK
    using Microsoft.EntityFrameworkCore;  // ✓ OK (infrastructure concern)
    
    public class OrderRepository : IOrderRepository  // Implements domain interface
    {
        // EF implementation
    }
}

// ❌ BAD: Dependencies point outward
namespace Domain
{
    using Infrastructure;  // ✗ WRONG! Domain depends on infrastructure
    using Microsoft.EntityFrameworkCore;  // ✗ WRONG! Domain depends on EF
    
    public class Order : DbContext  // ✗ WRONG! Domain inherits infrastructure
    {
    }
}
```

## Why It's a Problem

1. **Hard to test**: Must mock infrastructure to test business logic
2. **Coupling**: Business rules tied to framework versions
3. **Poor understanding**: Business logic hidden in technical code
4. **Inflexibility**: Changing infrastructure requires changing domain

## Symptoms

- Domain classes inherit from `DbContext`, `Controller`, etc.
- EF attributes (`[Table]`, `[Column]`) in domain models
- HTTP calls, logging, or file I/O in domain logic
- Can't test without mocking multiple infrastructure concerns
- Using `async`/`await` for pure calculations

## Benefits

- **Testability**: Pure functions easy to test
- **Flexibility**: Swap infrastructure without touching domain
- **Clarity**: Business rules stand out clearly
- **Maintainability**: Changes isolated to appropriate layer
- **Portability**: Domain logic works anywhere

## See Also

- [Clean Architecture](./clean-architecture.md) — architectural layers
- [Dependency Rule Enforcement](./dependency-rule-enforcement.md) — compile-time checks
- [Repository Pattern](./repository-pattern.md) — data access abstraction
- [Boundary Enforcement (DTO vs. Domain)](./dto-domain-boundary.md) — separating concerns
- [Anti-Corruption Layer](./anti-corruption-layer.md) — protecting domain boundaries
