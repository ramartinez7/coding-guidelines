# Application-Specific Business Rules

> Application layer can contain application-specific business rules (workflow orchestration, external system coordination) distinct from core domain rules—understanding this distinction prevents all logic from being pushed into the domain or scattered across controllers.

## Problem

Not all business logic belongs in the domain. Some rules exist purely because of application-level concerns (external system integration, multi-step workflows, transaction boundaries). When these are forced into the domain, it becomes polluted with infrastructure concerns. When left in controllers, they're scattered and hard to test.

## Core Concept

**Three Types of Business Logic:**

1. **Core Domain Rules** (Domain Layer)
   - Fundamental business invariants
   - Would exist even if there were no application
   - Example: "Order total must be sum of item prices"

2. **Application-Specific Rules** (Application Layer)
   - Orchestration and coordination logic
   - Multi-aggregate workflows
   - External system integration rules
   - Example: "Reserve inventory before placing order"

3. **Input Validation** (Application Layer)
   - Format validation, required fields
   - Not core business rules
   - Example: "Email must match regex pattern"

## Example

### ❌ Before (Confused Responsibilities)

```csharp
// ❌ Application orchestration in domain
public class Order
{
    private readonly IInventoryService _inventoryService;  // ❌ Infrastructure in domain
    
    public async Task Submit()  // ❌ Async in domain (usually infrastructure concern)
    {
        Status = OrderStatus.Submitted;
        
        // ❌ External system coordination in domain
        await _inventoryService.ReserveStockAsync(Items);
    }
}

// ❌ Core business rules in application layer
public class PlaceOrderUseCase
{
    public async Task ExecuteAsync(PlaceOrderCommand command)
    {
        var order = Order.Create(command.CustomerId);
        
        // ❌ Business rule in application layer
        if (order.Items.Sum(i => i.Quantity) > 100)
            throw new InvalidOperationException("Cannot order more than 100 items");
        
        // ❌ Calculation in application layer
        var total = order.Items.Sum(i => i.Price * i.Quantity);
        order.SetTotal(total);
        
        await _orderRepository.SaveAsync(order);
    }
}
```

### ✅ After (Clear Separation)

```csharp
// Domain/Entities/Order.cs (Core Domain Rules)
namespace MyApp.Domain.Entities
{
    /// <summary>
    /// Domain entity - contains ONLY core business rules.
    /// </summary>
    public sealed class Order
    {
        private readonly List<OrderItem> _items = new();
        
        public OrderId Id { get; }
        public CustomerId CustomerId { get; }
        public Money Total { get; private set; }
        public OrderStatus Status { get; private set; }
        
        // ✅ Core domain rule - would exist even without an application
        public Result<Unit, string> AddItem(Product product, int quantity)
        {
            if (quantity <= 0)
                return Result<Unit, string>.Failure(
                    "Quantity must be positive");  // Core invariant
            
            // ✅ Core business rule
            if (_items.Sum(i => i.Quantity) + quantity > 100)
                return Result<Unit, string>.Failure(
                    "Cannot order more than 100 items total");
            
            var item = new OrderItem(product.Id, quantity, product.Price);
            _items.Add(item);
            
            // ✅ Core domain logic - calculation
            RecalculateTotal();
            
            return Result<Unit, string>.Success(Unit.Value);
        }
        
        // ✅ Core domain rule - state transition
        public Result<Unit, string> Submit()
        {
            if (_items.Count == 0)
                return Result<Unit, string>.Failure(
                    "Cannot submit empty order");  // Core invariant
            
            if (Status != OrderStatus.Draft)
                return Result<Unit, string>.Failure(
                    "Order already submitted");
            
            Status = OrderStatus.Submitted;
            return Result<Unit, string>.Success(Unit.Value);
        }
        
        private void RecalculateTotal()
        {
            Total = _items.Select(i => i.Total)
                          .Aggregate(Money.Zero, (sum, price) => sum + price);
        }
    }
}

// Application/Commands/PlaceOrder/PlaceOrderCommandHandler.cs (Application Rules)
namespace MyApp.Application.Commands.PlaceOrder
{
    /// <summary>
    /// Application service - contains application-specific orchestration rules.
    /// </summary>
    public sealed class PlaceOrderCommandHandler
    {
        private readonly IOrderRepository _orderRepository;
        private readonly IProductRepository _productRepository;
        private readonly IInventoryService _inventoryService;
        private readonly IPaymentGateway _paymentGateway;
        private readonly IEmailService _emailService;
        private readonly IUnitOfWork _unitOfWork;
        
        public async Task<Result<OrderId, PlaceOrderError>> HandleAsync(
            PlaceOrderCommand command)
        {
            // ✅ Application rule: fetch dependencies
            var customer = await _customerRepository.GetByIdAsync(command.CustomerId);
            if (customer == null)
                return Result.Failure(new PlaceOrderError.CustomerNotFound());
            
            // ✅ Core domain: create order
            var order = Order.Create(command.CustomerId);
            
            // ✅ Core domain: add items (business rules in domain)
            foreach (var itemCmd in command.Items)
            {
                var product = await _productRepository.GetByIdAsync(itemCmd.ProductId);
                
                var addResult = await product.Match(
                    onSome: p => Task.FromResult(order.AddItem(p, itemCmd.Quantity)),
                    onNone: () => Task.FromResult(
                        Result<Unit, string>.Failure("Product not found")));
                
                if (!addResult.IsSuccess)
                    return Result.Failure(new PlaceOrderError.InvalidItem(addResult.Error));
            }
            
            // ✅ Application rule: Reserve inventory BEFORE submitting order
            // This is application-specific - coordinates external system with domain
            foreach (var item in order.Items)
            {
                var reserveResult = await _inventoryService.ReserveStockAsync(
                    item.ProductId,
                    item.Quantity);
                
                if (!reserveResult.IsSuccess)
                {
                    // ✅ Application rule: rollback inventory on failure
                    await _inventoryService.RollbackReservationsAsync(order.Items);
                    return Result.Failure(new PlaceOrderError.InsufficientStock());
                }
            }
            
            // ✅ Application rule: Process payment BEFORE submitting
            // Orchestration rule - coordinates payment gateway with domain
            if (order.Total.Amount > 0)
            {
                var paymentResult = await _paymentGateway.ProcessPaymentAsync(
                    order.Total,
                    command.PaymentMethod);
                
                if (!paymentResult.IsSuccess)
                {
                    // ✅ Application rule: rollback inventory if payment fails
                    await _inventoryService.RollbackReservationsAsync(order.Items);
                    return Result.Failure(new PlaceOrderError.PaymentFailed());
                }
            }
            
            // ✅ Core domain: submit order (business rule in domain)
            var submitResult = order.Submit();
            if (!submitResult.IsSuccess)
            {
                // ✅ Application rule: rollback external systems
                await _inventoryService.RollbackReservationsAsync(order.Items);
                await _paymentGateway.RefundAsync(paymentId);
                return Result.Failure(new PlaceOrderError.SubmitFailed(submitResult.Error));
            }
            
            // ✅ Application rule: transaction boundary
            await _orderRepository.SaveAsync(order);
            await _unitOfWork.CommitAsync();
            
            // ✅ Application rule: send notification after commit
            await _emailService.SendOrderConfirmationAsync(order.Id, customer.Email);
            
            return Result.Success(order.Id);
        }
    }
}
```

## Rules Classification

| Concern | Layer | Example |
|---------|-------|---------|
| **Core Invariants** | Domain | "Order total = sum of items", "Quantity > 0" |
| **State Transitions** | Domain | "Can only cancel Draft or Submitted orders" |
| **Calculations** | Domain | "Calculate discount based on customer tier" |
| **Workflow Orchestration** | Application | "Reserve inventory → Process payment → Submit order" |
| **External System Coordination** | Application | "If payment fails, rollback inventory" |
| **Transaction Boundaries** | Application | "Commit after all validations pass" |
| **Multi-Aggregate Rules** | Application | "Check customer credit limit before placing order" |
| **Input Validation** | Application | "Email must be valid format" |

## Application-Specific Rules Examples

### 1. Workflow Sequencing

```csharp
// ✅ Application rule: Order of operations matters
public async Task<Result<OrderId, PlaceOrderError>> HandleAsync(PlaceOrderCommand command)
{
    // Step 1: Reserve inventory
    var reserveResult = await _inventoryService.ReserveStockAsync(items);
    if (!reserveResult.IsSuccess)
        return Result.Failure(/* ... */);
    
    // Step 2: Process payment
    var paymentResult = await _paymentGateway.ProcessPaymentAsync(amount);
    if (!paymentResult.IsSuccess)
    {
        await _inventoryService.RollbackReservationsAsync(items);  // Compensate
        return Result.Failure(/* ... */);
    }
    
    // Step 3: Submit order
    var submitResult = order.Submit();
    // ...
}
```

### 2. Multi-Aggregate Coordination

```csharp
// ✅ Application rule: Check customer credit limit (spans Customer and Order aggregates)
public async Task<Result<OrderId, PlaceOrderError>> HandleAsync(PlaceOrderCommand command)
{
    var customer = await _customerRepository.GetByIdAsync(command.CustomerId);
    var order = Order.Create(command.CustomerId);
    
    // ✅ Application-specific rule crossing aggregate boundaries
    if (customer.OutstandingBalance + order.Total > customer.CreditLimit)
        return Result.Failure(new PlaceOrderError.CreditLimitExceeded());
    
    // ...
}
```

### 3. External System Constraints

```csharp
// ✅ Application rule: Fraud detection via external service
public async Task<Result<OrderId, PlaceOrderError>> HandleAsync(PlaceOrderCommand command)
{
    var order = Order.Create(command.CustomerId);
    
    // ✅ Application-specific: consult external fraud detection service
    var fraudCheckResult = await _fraudDetectionService.AnalyzeOrderAsync(order);
    
    if (fraudCheckResult.IsHighRisk)
    {
        order.MarkForReview();  // Domain operation
        await _emailService.NotifyFraudTeamAsync(order.Id);  // Application concern
    }
    
    // ...
}
```

## Benefits

1. **Clear Separation**: Know where each type of rule belongs
2. **Testability**: Domain rules testable without infrastructure; application rules testable with fakes
3. **Reusability**: Core domain rules work regardless of workflow
4. **Maintainability**: Rules organized by responsibility

## Common Mistakes

❌ **Pushing All Logic to Domain**
```csharp
// ❌ Domain handling external systems
public class Order
{
    public async Task Submit(IInventoryService inventory, IPaymentGateway payment)
    {
        await inventory.ReserveStockAsync(Items);  // ❌ External coordination
        await payment.ProcessPaymentAsync(Total);   // ❌ External coordination
        Status = OrderStatus.Submitted;
    }
}
```

❌ **Leaving Application Rules in Controllers**
```csharp
// ❌ Orchestration in controller
[HttpPost]
public async Task<IActionResult> PlaceOrder(PlaceOrderRequest request)
{
    var order = Order.Create(request.CustomerId);
    
    await _inventoryService.ReserveStockAsync(order.Items);  // ❌ Workflow in controller
    await _paymentGateway.ProcessPaymentAsync(order.Total);  // ❌ Workflow in controller
    
    await _orderRepository.SaveAsync(order);
    return Ok();
}
```

## See Also

- [Domain Service vs Application Service](./domain-service-vs-application-service.md) — service layer distinction
- [Use Cases](./use-cases.md) — application service pattern
- [Application Service Layer Isolation](./application-service-isolation.md) — orchestration layer
- [Domain Invariants](./domain-invariants.md) — enforcing core rules
