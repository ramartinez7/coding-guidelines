# Domain Services (Stateless Domain Operations)

> Business logic that doesn't naturally belong to any single entity or value object—use domain services to coordinate operations across multiple aggregates.

## Problem

Not all business logic fits cleanly into entities. Operations that involve multiple aggregates, complex calculations, or domain-wide policies need a home. Putting them in entities creates artificial coupling or "god objects."

## Example

### ❌ Before

```csharp
// Business logic awkwardly stuffed into entity
public class Order
{
    private readonly IProductRepository _productRepo;  // Entity depends on repository!
    private readonly IPricingRepository _pricingRepo;
    private readonly ITaxCalculator _taxCalculator;
    
    public Order(
        IProductRepository productRepo,
        IPricingRepository pricingRepo,
        ITaxCalculator taxCalculator)
    {
        _productRepo = productRepo;
        _pricingRepo = pricingRepo;
        _taxCalculator = taxCalculator;
    }
    
    // Logic that needs multiple aggregates doesn't belong in entity
    public async Task<decimal> CalculateTotalWithDiscounts()
    {
        decimal subtotal = 0;
        
        foreach (var item in Items)
        {
            // Entity loading other entities—bad!
            var product = await _productRepo.GetByIdAsync(item.ProductId);
            var pricing = await _pricingRepo.GetCurrentPricingAsync(item.ProductId);
            
            var price = pricing.CurrentPrice;
            
            // Complex discount logic in entity
            if (pricing.HasActivePromotion)
                price = price * (1 - pricing.DiscountPercentage);
            
            if (CustomerId.IsPremiumMember)
                price = price * 0.95m;
            
            subtotal += price * item.Quantity;
        }
        
        var tax = _taxCalculator.Calculate(subtotal, ShippingAddress.State);
        return subtotal + tax;
    }
}
```

**Problems:**
- Entity depends on repositories and services
- Business logic spanning multiple aggregates in one entity
- Hard to test
- Violates aggregate boundaries
- Entity becomes a "god object"

### ✅ After

```csharp
// ============================================
// DOMAIN ENTITIES (Pure, no external dependencies)
// ============================================
namespace Orders.Domain
{
    /// <summary>
    /// Entity focused on its own consistency.
    /// </summary>
    public sealed class Order
    {
        private readonly List<OrderLine> _lines = new();
        
        public OrderId Id { get; }
        public CustomerId CustomerId { get; }
        public Address ShippingAddress { get; private set; }
        public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
        
        // Simple calculation that only needs data in this aggregate
        public Money Subtotal => _lines
            .Select(line => line.LineTotal)
            .Aggregate(Money.Zero, (sum, total) => sum + total);
        
        public void AddLine(ProductId productId, int quantity, Money unitPrice)
        {
            var line = new OrderLine(productId, quantity, unitPrice);
            _lines.Add(line);
        }
    }
    
    public sealed record OrderLine(ProductId ProductId, int Quantity, Money UnitPrice)
    {
        public Money LineTotal => UnitPrice * Quantity;
    }
}

namespace Catalog.Domain
{
    /// <summary>
    /// Product aggregate—independent of Order.
    /// </summary>
    public sealed class Product
    {
        public ProductId Id { get; }
        public string Name { get; }
        public Money BasePrice { get; private set; }
        
        public void UpdatePrice(Money newPrice)
        {
            BasePrice = newPrice;
        }
    }
}

namespace Pricing.Domain
{
    /// <summary>
    /// Pricing aggregate—owns discount logic for products.
    /// </summary>
    public sealed class ProductPricing
    {
        public ProductId ProductId { get; }
        public Money CurrentPrice { get; private set; }
        public Option<Promotion> ActivePromotion { get; private set; }
        
        public Money CalculateEffectivePrice()
        {
            return ActivePromotion.Match(
                onSome: promo => promo.ApplyDiscount(CurrentPrice),
                onNone: () => CurrentPrice);
        }
    }
}

// ============================================
// DOMAIN SERVICE (Coordinates across aggregates)
// ============================================
namespace Orders.Domain.Services
{
    /// <summary>
    /// Domain Service: Encapsulates business logic that spans multiple aggregates.
    /// Stateless—no identity, no lifecycle, just operations.
    /// </summary>
    public sealed class OrderPricingService
    {
        private readonly IProductRepository _productRepo;
        private readonly IPricingRepository _pricingRepo;
        private readonly ITaxCalculator _taxCalculator;
        private readonly ICustomerDiscountPolicy _discountPolicy;
        
        public OrderPricingService(
            IProductRepository productRepo,
            IPricingRepository pricingRepo,
            ITaxCalculator taxCalculator,
            ICustomerDiscountPolicy discountPolicy)
        {
            _productRepo = productRepo;
            _pricingRepo = pricingRepo;
            _taxCalculator = taxCalculator;
            _discountPolicy = discountPolicy;
        }
        
        /// <summary>
        /// Calculates order total with all discounts and taxes.
        /// Domain logic coordinating multiple aggregates.
        /// </summary>
        public async Task<Result<OrderTotal, string>> CalculateOrderTotal(Order order)
        {
            Money subtotal = Money.Zero;
            
            foreach (var line in order.Lines)
            {
                // Load pricing aggregate
                var pricingOpt = await _pricingRepo.GetByProductIdAsync(line.ProductId);
                
                if (pricingOpt.IsNone)
                    return Result<OrderTotal, string>.Failure(
                        $"Pricing not found for product {line.ProductId}");
                
                var pricing = pricingOpt.Value;
                
                // Use domain logic from pricing aggregate
                var effectivePrice = pricing.CalculateEffectivePrice();
                
                // Apply customer-specific discounts (domain policy)
                effectivePrice = await _discountPolicy.ApplyCustomerDiscount(
                    order.CustomerId,
                    effectivePrice);
                
                var lineTotal = effectivePrice * line.Quantity;
                subtotal = subtotal + lineTotal;
            }
            
            // Calculate tax (domain logic)
            var tax = _taxCalculator.Calculate(subtotal, order.ShippingAddress);
            var total = subtotal + tax;
            
            return Result<OrderTotal, string>.Success(
                new OrderTotal(subtotal, tax, total));
        }
    }
    
    public sealed record OrderTotal(Money Subtotal, Money Tax, Money Total);
}
```

## Domain Service Characteristics

```csharp
/// <summary>
/// Domain Service checklist:
/// ✓ Stateless—no internal state
/// ✓ No identity—not an entity
/// ✓ Domain logic—not infrastructure
/// ✓ Coordinates aggregates—doesn't modify them directly
/// </summary>
public interface IDomainService { }

// ✅ Good domain service
public sealed class TransferService : IDomainService
{
    public Result<Unit, string> TransferFunds(
        BankAccount fromAccount,
        BankAccount toAccount,
        Money amount)
    {
        // Coordinates two aggregates
        var withdrawResult = fromAccount.Withdraw(amount);
        if (!withdrawResult.IsSuccess)
            return withdrawResult;
        
        var depositResult = toAccount.Deposit(amount);
        if (!depositResult.IsSuccess)
        {
            // Rollback if needed
            fromAccount.Deposit(amount);
            return depositResult;
        }
        
        return Result<Unit, string>.Success(Unit.Value);
    }
}

// ❌ Not a domain service (has state)
public class OrderManager  // Has state, not a service
{
    private List<Order> _pendingOrders = new();  // State!
    
    public void AddPendingOrder(Order order)
    {
        _pendingOrders.Add(order);
    }
}

// ✅ Good domain service (stateless)
public sealed class OrderValidationService : IDomainService
{
    public Result<Unit, string> ValidateOrder(Order order, Customer customer)
    {
        // No internal state, just operations
        if (!customer.IsActive)
            return Result<Unit, string>.Failure("Customer not active");
        
        if (order.Lines.Count == 0)
            return Result<Unit, string>.Failure("Order has no items");
        
        if (order.Subtotal.Amount < 0)
            return Result<Unit, string>.Failure("Order total cannot be negative");
        
        return Result<Unit, string>.Success(Unit.Value);
    }
}
```

## Domain Service vs Application Service

```csharp
// ============================================
// DOMAIN SERVICE (Business logic)
// ============================================
namespace Orders.Domain.Services
{
    /// <summary>
    /// Domain Service: Pure business logic, no infrastructure concerns.
    /// </summary>
    public sealed class ShipmentService
    {
        private readonly IInventoryRepository _inventoryRepo;
        
        /// <summary>
        /// Business rule: Can only ship if all items are in stock.
        /// </summary>
        public async Task<Result<ShipmentPlan, string>> CreateShipmentPlan(Order order)
        {
            var plan = new ShipmentPlan(order.Id);
            
            foreach (var line in order.Lines)
            {
                var inventoryOpt = await _inventoryRepo.GetByProductIdAsync(line.ProductId);
                
                if (inventoryOpt.IsNone)
                    return Result<ShipmentPlan, string>.Failure(
                        $"Product {line.ProductId} not found in inventory");
                
                var inventory = inventoryOpt.Value;
                
                // Domain logic: check availability
                if (inventory.AvailableQuantity < line.Quantity)
                    return Result<ShipmentPlan, string>.Failure(
                        $"Insufficient stock for {line.ProductId}");
                
                plan.AddItem(line.ProductId, line.Quantity, inventory.WarehouseLocation);
            }
            
            return Result<ShipmentPlan, string>.Success(plan);
        }
    }
}

// ============================================
// APPLICATION SERVICE (Orchestration)
// ============================================
namespace Orders.Application
{
    /// <summary>
    /// Application Service: Orchestrates domain services, repositories, and events.
    /// Handles transaction boundaries and infrastructure concerns.
    /// </summary>
    public sealed class ShipOrderHandler
    {
        private readonly IOrderRepository _orderRepo;
        private readonly ShipmentService _shipmentService;
        private readonly IShippingProvider _shippingProvider;
        private readonly IEventDispatcher _eventDispatcher;
        
        public async Task<Result<Unit, string>> Handle(ShipOrderCommand command)
        {
            // Load aggregate
            var orderOpt = await _orderRepo.GetByIdAsync(command.OrderId);
            if (orderOpt.IsNone)
                return Result<Unit, string>.Failure("Order not found");
            
            var order = orderOpt.Value;
            
            // Use domain service for business logic
            var planResult = await _shipmentService.CreateShipmentPlan(order);
            if (!planResult.IsSuccess)
                return Result<Unit, string>.Failure(planResult.Error);
            
            var plan = planResult.Value;
            
            // Infrastructure: Call external shipping provider
            var shipmentResult = await _shippingProvider.CreateShipment(plan);
            if (!shipmentResult.IsSuccess)
                return Result<Unit, string>.Failure(shipmentResult.Error);
            
            var shipment = shipmentResult.Value;
            
            // Update aggregate
            order.MarkAsShipped(shipment.TrackingNumber, shipment.ShippedAt);
            
            // Save and publish events (infrastructure concern)
            await _orderRepo.SaveAsync(order);
            await _eventDispatcher.DispatchAsync(new OrderShippedEvent(
                order.Id,
                shipment.TrackingNumber,
                shipment.ShippedAt));
            
            return Result<Unit, string>.Success(Unit.Value);
        }
    }
}
```

## When to Use Domain Services

### Use Domain Service When:

- **Logic spans multiple aggregates**
  ```csharp
  // Transferring funds between two accounts
  public class TransferService
  {
      public Result<Unit, string> Transfer(
          BankAccount from, BankAccount to, Money amount) { }
  }
  ```

- **Complex calculation requires multiple domain objects**
  ```csharp
  // Calculating insurance premium based on customer, vehicle, and coverage
  public class InsurancePremiumCalculator
  {
      public Money Calculate(Customer customer, Vehicle vehicle, Coverage coverage) { }
  }
  ```

- **Domain policy doesn't belong to any single aggregate**
  ```csharp
  // Credit approval policy checking customer history and loan terms
  public class CreditApprovalService
  {
      public Result<CreditDecision, string> EvaluateApplication(
          Customer customer, LoanApplication application) { }
  }
  ```

- **Business rule coordination**
  ```csharp
  // Validating order against customer limits and product availability
  public class OrderValidationService
  {
      public Result<Unit, string> ValidateOrder(Order order, Customer customer) { }
  }
  ```

### Don't Use Domain Service When:

- **Logic belongs to single entity**
  ```csharp
  // ❌ Don't create service
  public class OrderTotalCalculator
  {
      public Money Calculate(Order order) => order.CalculateTotal();
  }
  
  // ✅ Put in entity
  public class Order
  {
      public Money CalculateTotal() { }
  }
  ```

- **It's infrastructure concern**
  ```csharp
  // ❌ Not domain service (infrastructure)
  public class EmailService { }
  
  // ✅ Application service or infrastructure service
  ```

## Domain Service Examples

```csharp
// Domain service: Pricing strategy
public sealed class DynamicPricingService
{
    public Money CalculatePrice(Product product, Customer customer, DateTimeOffset when)
    {
        var basePrice = product.BasePrice;
        
        // Time-based pricing (domain rule)
        if (IsOffPeakHours(when))
            basePrice = basePrice * 0.9m;
        
        // Customer tier pricing (domain rule)
        if (customer.Tier == CustomerTier.Premium)
            basePrice = basePrice * 0.95m;
        
        // Product category pricing (domain rule)
        if (product.Category.IsLuxury)
            basePrice = basePrice * 1.1m;
        
        return basePrice;
    }
    
    private bool IsOffPeakHours(DateTimeOffset when)
    {
        var hour = when.Hour;
        return hour >= 22 || hour < 6;  // Domain rule: 10 PM to 6 AM
    }
}

// Domain service: Inventory allocation
public sealed class InventoryAllocationService
{
    public Result<Allocation, string> AllocateInventory(
        Order order,
        List<Warehouse> warehouses)
    {
        var allocation = new Allocation(order.Id);
        
        foreach (var line in order.Lines)
        {
            // Domain logic: prefer closest warehouse
            var sortedWarehouses = warehouses
                .OrderBy(w => w.DistanceTo(order.ShippingAddress))
                .ToList();
            
            var remainingQuantity = line.Quantity;
            
            foreach (var warehouse in sortedWarehouses)
            {
                var available = warehouse.GetAvailableQuantity(line.ProductId);
                var toAllocate = Math.Min(available, remainingQuantity);
                
                if (toAllocate > 0)
                {
                    allocation.AddAllocation(warehouse.Id, line.ProductId, toAllocate);
                    remainingQuantity -= toAllocate;
                }
                
                if (remainingQuantity == 0)
                    break;
            }
            
            if (remainingQuantity > 0)
                return Result<Allocation, string>.Failure(
                    $"Cannot fulfill {remainingQuantity} units of {line.ProductId}");
        }
        
        return Result<Allocation, string>.Success(allocation);
    }
}

// Domain service: Business rule enforcement
public sealed class LoanEligibilityService
{
    public Result<LoanApproval, string> CheckEligibility(
        Customer customer,
        LoanApplication application)
    {
        // Domain rule: minimum credit score
        if (customer.CreditScore < 650)
            return Result<LoanApproval, string>.Failure("Credit score too low");
        
        // Domain rule: debt-to-income ratio
        var monthlyPayment = CalculateMonthlyPayment(application);
        var debtToIncomeRatio = (customer.MonthlyDebt + monthlyPayment) / customer.MonthlyIncome;
        
        if (debtToIncomeRatio > 0.43m)
            return Result<LoanApproval, string>.Failure("Debt-to-income ratio too high");
        
        // Domain rule: employment history
        if (customer.EmploymentMonths < 24)
            return Result<LoanApproval, string>.Failure("Insufficient employment history");
        
        return Result<LoanApproval, string>.Success(
            new LoanApproval(application.Id, monthlyPayment));
    }
}
```

## Why It's a Problem

1. **God entities**: Entities with too many responsibilities
2. **Artificial coupling**: Logic forced into wrong aggregate
3. **Duplication**: Same logic repeated across entities
4. **Hard to test**: Complex dependencies in entities

## Symptoms

- Entities with many constructor dependencies
- Methods that load and modify multiple aggregates
- Complex business logic scattered across entities
- Difficulty deciding which entity owns certain logic

## Benefits

- **Clear responsibility**: Domain logic in appropriate place
- **Reusable**: Services used across multiple use cases
- **Testable**: Stateless services easy to test
- **Maintainable**: Business rules centralized

## See Also

- [Aggregate Roots](./aggregate-roots.md) — entity boundaries
- [Repository Pattern](./repository-pattern.md) — data access
- [Domain Events](./domain-events.md) — decoupling side effects
- [Specification Pattern](./specification-pattern.md) — business rules as types
