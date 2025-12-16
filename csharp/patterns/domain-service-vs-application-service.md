# Domain Service vs Application Service

> Domain services encapsulate business logic that doesn't belong to a single entity, while application services orchestrate use cases and coordinate between domain and infrastructure—understanding the difference prevents logic from landing in the wrong layer.

## Problem

Confusion between domain services and application services leads to business logic leaking into the application layer or orchestration code polluting the domain layer.

## Core Concepts

**Domain Service:**
- Contains business logic that doesn't naturally fit in an entity or value object
- Operations that span multiple aggregates
- Part of domain layer (pure business logic)
- No dependencies on infrastructure
- Example: `PricingService`, `FraudDetectionService`, `ShippingCostCalculator`

**Application Service:**
- Orchestrates use cases
- Coordinates domain objects and infrastructure
- Part of application layer
- Can depend on repositories, infrastructure services
- Example: `PlaceOrderUseCase`, `RegisterUserUseCase`

## Example

### ❌ Before (Confused Responsibilities)

```csharp
// ❌ Business logic in application service
public class PlaceOrderUseCase
{
    public async Task ExecuteAsync(PlaceOrderCommand command)
    {
        var order = Order.Create(command.CustomerId);
        
        // ❌ Pricing logic in application service (should be domain service)
        var discount = 0m;
        if (order.Total > 100)
            discount = order.Total * 0.1m;  // 10% discount
        if (command.CustomerId.IsPremium)
            discount += order.Total * 0.05m;  // additional 5%
        
        order.ApplyDiscount(discount);
        
        await _orderRepository.SaveAsync(order);
    }
}

// ❌ Orchestration in domain service
public class OrderDomainService
{
    private readonly IOrderRepository _orderRepository;  // ❌ Repository in domain
    
    public async Task PlaceOrder(Order order)
    {
        // ❌ Persistence concern in domain service
        await _orderRepository.SaveAsync(order);
        
        // ❌ External service call in domain service
        await _emailService.SendConfirmationAsync(order);
    }
}
```

### ✅ After (Clear Separation)

```csharp
// Domain/Services/PricingService.cs (Domain Service)
namespace MyApp.Domain.Services
{
    /// <summary>
    /// Domain service - pure business logic, no infrastructure.
    /// Calculates prices based on business rules.
    /// </summary>
    public sealed class PricingService
    {
        public Money CalculateDiscount(Order order, Customer customer)
        {
            var discount = Money.Zero;
            
            // ✅ Business rule: bulk discount
            if (order.Total.Amount > 100)
                discount += order.Total * 0.1m;
            
            // ✅ Business rule: premium customer discount
            if (customer.IsPremium)
                discount += order.Total * 0.05m;
            
            // ✅ Business rule: first order discount
            if (customer.OrderCount == 0)
                discount += Money.USD(10);
            
            return discount;
        }
        
        public Money CalculateShippingCost(Order order, Address destination)
        {
            // Business logic for shipping costs
            var weight = order.Items.Sum(i => i.Product.Weight * i.Quantity);
            var baseRate = Money.USD(5);
            
            if (weight > 10)
                baseRate += Money.USD(weight * 0.5m);
            
            if (destination.Country != "US")
                baseRate += Money.USD(15);
            
            return baseRate;
        }
    }
}

// Domain/Services/FraudDetectionService.cs (Domain Service)
namespace MyApp.Domain.Services
{
    /// <summary>
    /// Domain service for fraud detection logic.
    /// </summary>
    public sealed class FraudDetectionService
    {
        public bool IsSuspicious(Order order, Customer customer)
        {
            // ✅ Business rule: large first order is suspicious
            if (customer.OrderCount == 0 && order.Total.Amount > 1000)
                return true;
            
            // ✅ Business rule: rapid-fire orders
            if (customer.OrdersInLast24Hours > 10)
                return true;
            
            // ✅ Business rule: shipping to different country
            if (order.ShippingAddress.Country != customer.BillingAddress.Country)
                return true;
            
            return false;
        }
    }
}

// Application/UseCases/PlaceOrder/PlaceOrderUseCase.cs (Application Service)
namespace MyApp.Application.UseCases.PlaceOrder
{
    /// <summary>
    /// Application service - orchestrates domain logic and infrastructure.
    /// </summary>
    public sealed class PlaceOrderUseCase
    {
        private readonly IOrderRepository _orderRepository;
        private readonly ICustomerRepository _customerRepository;
        private readonly IUnitOfWork _unitOfWork;
        private readonly PricingService _pricingService;  // ✅ Domain service
        private readonly FraudDetectionService _fraudService;  // ✅ Domain service
        private readonly IEmailService _emailService;  // ✅ Infrastructure service
        
        public PlaceOrderUseCase(
            IOrderRepository orderRepository,
            ICustomerRepository customerRepository,
            IUnitOfWork unitOfWork,
            PricingService pricingService,
            FraudDetectionService fraudService,
            IEmailService emailService)
        {
            _orderRepository = orderRepository;
            _customerRepository = customerRepository;
            _unitOfWork = unitOfWork;
            _pricingService = pricingService;
            _fraudService = fraudService;
            _emailService = emailService;
        }
        
        public async Task<Result<OrderId, PlaceOrderError>> ExecuteAsync(
            PlaceOrderCommand command)
        {
            // ✅ Application service: fetch dependencies
            var customer = await _customerRepository.GetByIdAsync(command.CustomerId);
            if (customer == null)
                return Result.Failure(new PlaceOrderError.CustomerNotFound());
            
            // ✅ Domain logic: create order
            var order = Order.Create(command.CustomerId);
            
            // ✅ Domain service: calculate discount
            var discount = _pricingService.CalculateDiscount(order, customer);
            order.ApplyDiscount(discount);
            
            // ✅ Domain service: fraud detection
            if (_fraudService.IsSuspicious(order, customer))
            {
                order.MarkAsRequiresReview();
            }
            else
            {
                order.Submit();
            }
            
            // ✅ Application service: persist
            await _orderRepository.SaveAsync(order);
            await _unitOfWork.CommitAsync();
            
            // ✅ Application service: side effects
            await _emailService.SendOrderConfirmationAsync(order.Id, customer.Email);
            
            return Result.Success(order.Id);
        }
    }
}

// DI Registration
services.AddScoped<PricingService>();  // Domain service
services.AddScoped<FraudDetectionService>();  // Domain service
services.AddScoped<PlaceOrderUseCase>();  // Application service
```

## Decision Matrix

| Concern | Domain Service | Application Service |
|---------|----------------|---------------------|
| **Business rules** | ✅ Yes | ❌ No |
| **Orchestration** | ❌ No | ✅ Yes |
| **Repository access** | ❌ No | ✅ Yes |
| **External services** | ❌ No | ✅ Yes |
| **Transaction management** | ❌ No | ✅ Yes |
| **Cross-aggregate logic** | ✅ Yes | ❌ No |
| **Calculations** | ✅ Yes | ❌ No |
| **Validation** | ✅ Yes | ❌ No (input validation only) |

## When to Use Domain Services

Use a domain service when business logic:
- Spans multiple aggregates
- Doesn't naturally belong to any single entity
- Requires coordination between entities
- Implements complex business rules

Examples:
- `MoneyTransferService` (involves two accounts)
- `PricingService` (considers product, customer, promotions)
- `RiskAssessmentService` (analyzes multiple factors)

## Benefits

1. **Clear Boundaries**: Business logic stays in domain
2. **Testability**: Domain services testable without infrastructure
3. **Reusability**: Same domain service used by multiple use cases
4. **Single Responsibility**: Each service has one clear purpose

## See Also

- [Clean Architecture](./clean-architecture.md) — layer separation
- [Use Cases](./use-cases.md) — application services
- [Domain Services](./domain-services.md) — existing pattern
- [Rich Domain Models](./rich-domain-models.md) — behavior in entities
