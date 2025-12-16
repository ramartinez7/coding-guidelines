# Onion Layer Dependencies

> Organize code in concentric layers where dependencies point inward toward the domain—outer layers depend on inner layers, never the reverse, ensuring the domain remains pure and infrastructure swappable.

## Problem

When dependencies flow in multiple directions, the architecture becomes tangled, making it impossible to test the domain in isolation or swap infrastructure.

## Core Concept

**Onion Architecture** (also called Clean Architecture or Hexagonal Architecture) organizes code in concentric layers:

```
┌─────────────────────────────────────┐
│    Infrastructure Layer             │ ← Outermost (depends on all inner layers)
│  ┌─────────────────────────────┐    │
│  │  Application Layer          │    │ ← Depends on Domain
│  │ ┌───────────────────────┐   │    │
│  │ │   Domain Layer        │   │    │ ← Innermost (depends on nothing)
│  │ │  (Pure Business)      │   │    │
│  │ └───────────────────────┘   │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘

Dependencies ALWAYS point inward →
```

## Layers and Dependencies

### Domain Layer (Core - No Dependencies)

```csharp
// Domain/Entities/Order.cs
namespace MyApp.Domain.Entities
{
    /// <summary>
    /// Pure domain entity - NO dependencies on outer layers.
    /// </summary>
    public sealed class Order
    {
        private readonly List<OrderItem> _items = new();
        
        public OrderId Id { get; }
        public CustomerId CustomerId { get; }
        public Money Total { get; private set; }
        
        // ✅ Domain depends only on other domain concepts
        public Result<Unit, string> AddItem(Product product, int quantity)
        {
            if (quantity <= 0)
                return Result<Unit, string>.Failure("Quantity must be positive");
            
            var item = new OrderItem(product.Id, quantity, product.Price);
            _items.Add(item);
            RecalculateTotal();
            
            return Result<Unit, string>.Success(Unit.Value);
        }
        
        private void RecalculateTotal()
        {
            Total = _items.Select(i => i.Total)
                          .Aggregate(Money.Zero, (sum, price) => sum + price);
        }
    }
}

// Domain/Services/PricingService.cs
namespace MyApp.Domain.Services
{
    /// <summary>
    /// Domain service - NO infrastructure dependencies.
    /// </summary>
    public sealed class PricingService
    {
        // ✅ Pure business logic
        public Money CalculateDiscount(Order order, Customer customer)
        {
            var discount = Money.Zero;
            
            if (order.Total.Amount > 100)
                discount += order.Total * 0.1m;
            
            if (customer.IsPremium)
                discount += order.Total * 0.05m;
            
            return discount;
        }
    }
}
```

### Application Layer (Depends on Domain Only)

```csharp
// Application/Commands/PlaceOrder/PlaceOrderCommandHandler.cs
namespace MyApp.Application.Commands.PlaceOrder
{
    /// <summary>
    /// Application service - depends on Domain and defines ports.
    /// Does NOT depend on Infrastructure.
    /// </summary>
    public sealed class PlaceOrderCommandHandler
    {
        // ✅ Depends on domain entities
        // ✅ Depends on ports (interfaces defined in Domain/Application)
        private readonly IOrderRepository _orderRepository;  // Port
        private readonly IProductRepository _productRepository;  // Port
        private readonly PricingService _pricingService;  // Domain service
        private readonly IUnitOfWork _unitOfWork;  // Port
        
        public PlaceOrderCommandHandler(
            IOrderRepository orderRepository,
            IProductRepository productRepository,
            PricingService pricingService,
            IUnitOfWork unitOfWork)
        {
            _orderRepository = orderRepository;
            _productRepository = productRepository;
            _pricingService = pricingService;
            _unitOfWork = unitOfWork;
        }
        
        public async Task<Result<OrderId, PlaceOrderError>> HandleAsync(
            PlaceOrderCommand command)
        {
            // ✅ Orchestrates domain logic
            var order = Order.Create(command.CustomerId);
            
            // ✅ Calls domain service
            var discount = _pricingService.CalculateDiscount(order, customer);
            order.ApplyDiscount(discount);
            
            // ✅ Uses ports (not concrete infrastructure)
            await _orderRepository.SaveAsync(order);
            await _unitOfWork.CommitAsync();
            
            return Result<OrderId, PlaceOrderError>.Success(order.Id);
        }
    }
}
```

### Infrastructure Layer (Depends on Application and Domain)

```csharp
// Infrastructure/Persistence/OrderRepository.cs
namespace MyApp.Infrastructure.Persistence
{
    /// <summary>
    /// Infrastructure implementation - depends on Domain.
    /// Implements port defined in Domain layer.
    /// </summary>
    public sealed class OrderRepository : IOrderRepository  // ✅ Implements domain port
    {
        private readonly MyDbContext _dbContext;
        
        public OrderRepository(MyDbContext dbContext)
        {
            _dbContext = dbContext;
        }
        
        public async Task<Option<Order>> GetByIdAsync(OrderId id)
        {
            // ✅ Infrastructure calls inward to Domain
            var entity = await _dbContext.Orders.FindAsync(id.Value);
            
            if (entity == null)
                return Option<Order>.None;
            
            var domainOrder = MapToDomain(entity);  // ✅ Creates domain object
            return Option<Order>.Some(domainOrder);
        }
        
        public async Task SaveAsync(Order order)  // ✅ Accepts domain object
        {
            var entity = MapToEntity(order);  // ✅ Maps to persistence model
            _dbContext.Orders.Update(entity);
        }
    }
}
```

## Dependency Rule Violations (Anti-Patterns)

### ❌ Domain Depending on Infrastructure

```csharp
// ❌ WRONG - Domain depends on EF Core
using Microsoft.EntityFrameworkCore;  // ❌ Infrastructure reference

[Table("Orders")]  // ❌ EF attribute in domain
public class Order
{
    [Key]  // ❌ Infrastructure concern
    public Guid Id { get; set; }
    
    public virtual Customer Customer { get; set; }  // ❌ EF lazy loading
}
```

### ❌ Domain Depending on Application

```csharp
// ❌ WRONG - Domain depends on Application
using MyApp.Application.Commands;  // ❌ Wrong direction

public class Order
{
    public void Submit(PlaceOrderCommand command)  // ❌ Domain knows about application
    {
        // ...
    }
}
```

### ✅ Correct: Application Depending on Domain

```csharp
// ✅ CORRECT - Application depends on Domain
using MyApp.Domain.Entities;  // ✅ Correct direction
using MyApp.Domain.Services;  // ✅ Correct direction

public class PlaceOrderCommandHandler
{
    public async Task<Result<OrderId, PlaceOrderError>> HandleAsync(
        PlaceOrderCommand command)
    {
        var order = Order.Create(command.CustomerId);  // ✅ Uses domain
        // ...
    }
}
```

## Project Structure

```
Solution: MyApp
├── MyApp.Domain/                    ← No dependencies
│   ├── Entities/
│   │   ├── Order.cs
│   │   └── Product.cs
│   ├── ValueObjects/
│   │   ├── OrderId.cs
│   │   └── Money.cs
│   ├── Services/
│   │   └── PricingService.cs
│   └── Ports/                       ← Interfaces
│       └── IOrderRepository.cs
│
├── MyApp.Application/               ← Depends on Domain only
│   ├── Commands/
│   │   └── PlaceOrder/
│   │       ├── PlaceOrderCommand.cs
│   │       └── PlaceOrderCommandHandler.cs
│   ├── Queries/
│   └── Ports/                       ← More interfaces
│       ├── IEmailService.cs
│       └── IUnitOfWork.cs
│
├── MyApp.Infrastructure/            ← Depends on Application & Domain
│   ├── Persistence/
│   │   ├── MyDbContext.cs
│   │   └── OrderRepository.cs       ← Implements IOrderRepository
│   ├── Email/
│   │   └── SendGridEmailAdapter.cs  ← Implements IEmailService
│   └── DependencyInjection.cs
│
└── MyApp.WebApi/                    ← Depends on all (composition root)
    ├── Controllers/
    │   └── OrdersController.cs
    ├── Program.cs                   ← Wires everything together
    └── appsettings.json
```

## Verifying Dependency Direction

Use `ArchUnitNET` or similar tools to enforce dependency rules:

```csharp
// Tests/Architecture/DependencyTests.cs
[Fact]
public void Domain_ShouldNotDependOnApplication()
{
    var domain = Types().That().ResideInNamespace("MyApp.Domain");
    var application = Types().That().ResideInNamespace("MyApp.Application");
    
    domain.Should().NotDependOnAny(application);
}

[Fact]
public void Domain_ShouldNotDependOnInfrastructure()
{
    var domain = Types().That().ResideInNamespace("MyApp.Domain");
    var infrastructure = Types().That().ResideInNamespace("MyApp.Infrastructure");
    
    domain.Should().NotDependOnAny(infrastructure);
}

[Fact]
public void Application_ShouldNotDependOnInfrastructure()
{
    var application = Types().That().ResideInNamespace("MyApp.Application");
    var infrastructure = Types().That().ResideInNamespace("MyApp.Infrastructure");
    
    application.Should().NotDependOnAny(infrastructure);
}
```

## Benefits

1. **Testability**: Inner layers testable without outer layers
2. **Flexibility**: Swap infrastructure without touching domain
3. **Clear Boundaries**: Dependency direction enforced
4. **Independence**: Domain free from framework coupling

## See Also

- [Clean Architecture](./clean-architecture.md) — detailed implementation
- [Dependency Rule Enforcement](./dependency-rule-enforcement.md) — automated checks
- [Port-Adapter Separation](./port-adapter-separation.md) — hexagonal architecture
- [Persistence Ignorance](./persistence-ignorance.md) — pure domain
