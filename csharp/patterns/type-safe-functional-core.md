# Type-Safe Functional Core (Functional Core, Imperative Shell)

> Business logic mixed with I/O and side effects makes testing hard and reasoning difficult—separate pure functional core from imperative shell using types to enforce the boundary.

## Problem

When business logic is intertwined with I/O operations, database calls, and other side effects, it becomes difficult to test, reason about, and maintain. The "Functional Core, Imperative Shell" pattern separates pure logic from effects, but without types to enforce the boundary, developers can accidentally mix them.

## Example

### ❌ Before

```csharp
// Business logic mixed with I/O
public class OrderProcessor
{
    private readonly IDatabase _db;
    private readonly IEmailService _email;
    private readonly ILogger _logger;
    
    public async Task<bool> ProcessOrder(int orderId)
    {
        // I/O: Database read
        var order = await _db.Orders.FindAsync(orderId);
        
        // I/O: Logging
        _logger.LogInformation($"Processing order {orderId}");
        
        // Business logic
        if (order.Total < 10)
        {
            _logger.LogWarning("Order too small");
            return false;
        }
        
        // More I/O: Database read
        var customer = await _db.Customers.FindAsync(order.CustomerId);
        
        // Business logic
        var discount = customer.IsVIP ? order.Total * 0.1m : 0;
        var finalTotal = order.Total - discount;
        
        // I/O: Database write
        order.FinalTotal = finalTotal;
        order.Status = "Processed";
        await _db.SaveChangesAsync();
        
        // I/O: Email
        await _email.SendAsync(customer.Email, "Order processed", $"Total: {finalTotal}");
        
        // I/O: Logging
        _logger.LogInformation($"Order {orderId} processed successfully");
        
        return true;
    }
}
```

**Problems:**
- Business logic and I/O completely intertwined
- Cannot test business logic without mocking database, email, and logging
- Difficult to understand what the core business rules are
- Easy to accidentally add I/O to business logic
- No way to run business logic in isolation

### ✅ After

```csharp
// ============================================
// FUNCTIONAL CORE (Pure, no I/O)
// ============================================
namespace Orders.Domain.Core
{
    /// <summary>
    /// Pure data structures—no I/O, no side effects.
    /// </summary>
    public sealed record OrderData(
        OrderId Id,
        CustomerId CustomerId,
        Money Total);
    
    public sealed record CustomerData(
        CustomerId Id,
        CustomerTier Tier);
    
    public enum CustomerTier
    {
        Standard,
        VIP
    }
    
    /// <summary>
    /// Pure business decisions—no I/O, just data transformation.
    /// </summary>
    public sealed record OrderDecision
    {
        private OrderDecision() { }
        
        public sealed record Accept(
            OrderId OrderId,
            Money OriginalTotal,
            Money Discount,
            Money FinalTotal) : OrderDecision;
        
        public sealed record Reject(
            OrderId OrderId,
            string Reason) : OrderDecision;
    }
    
    /// <summary>
    /// Pure function: Takes data, returns decision.
    /// No database, no I/O, no side effects.
    /// Easily testable with simple assertions.
    /// </summary>
    public static class OrderProcessingRules
    {
        private static readonly Money MinimumOrderAmount = Money.USD(10.00m);
        private static readonly decimal VipDiscountRate = 0.10m;
        
        public static OrderDecision ProcessOrder(OrderData order, CustomerData customer)
        {
            // Pure business logic
            if (order.Total.Amount < MinimumOrderAmount.Amount)
            {
                return new OrderDecision.Reject(
                    order.Id,
                    $"Order total ${order.Total.Amount} is below minimum ${MinimumOrderAmount.Amount}");
            }
            
            // Pure calculation
            var discount = CalculateDiscount(order.Total, customer.Tier);
            var finalTotal = Money.FromAmount(
                order.Total.Amount - discount.Amount,
                order.Total.Currency);
            
            return new OrderDecision.Accept(
                order.Id,
                order.Total,
                discount,
                finalTotal);
        }
        
        private static Money CalculateDiscount(Money total, CustomerTier tier)
        {
            var discountAmount = tier switch
            {
                CustomerTier.VIP => total.Amount * VipDiscountRate,
                CustomerTier.Standard => 0m,
                _ => 0m
            };
            
            return Money.FromAmount(discountAmount, total.Currency);
        }
    }
}

// ============================================
// IMPERATIVE SHELL (I/O, Side Effects)
// ============================================
namespace Orders.Application
{
    using Orders.Domain.Core;
    
    /// <summary>
    /// Commands and events—explicit about what I/O is needed.
    /// </summary>
    public sealed record ProcessOrderCommand(OrderId OrderId);
    
    public abstract record OrderProcessingEvent
    {
        public sealed record OrderProcessed(
            OrderId OrderId,
            Money FinalTotal) : OrderProcessingEvent;
        
        public sealed record OrderRejected(
            OrderId OrderId,
            string Reason) : OrderProcessingEvent;
    }
    
    /// <summary>
    /// Application service: Orchestrates I/O around pure core.
    /// Handles all side effects, delegates business logic to pure functions.
    /// </summary>
    public sealed class OrderProcessingService
    {
        private readonly IOrderRepository _orderRepo;
        private readonly ICustomerRepository _customerRepo;
        private readonly IEventPublisher _eventPublisher;
        
        public OrderProcessingService(
            IOrderRepository orderRepo,
            ICustomerRepository customerRepo,
            IEventPublisher eventPublisher)
        {
            _orderRepo = orderRepo;
            _customerRepo = customerRepo;
            _eventPublisher = eventPublisher;
        }
        
        public async Task<Result<OrderProcessingEvent, string>> Handle(
            ProcessOrderCommand command)
        {
            // I/O: Load data
            var orderOpt = await _orderRepo.GetByIdAsync(command.OrderId);
            if (orderOpt.IsNone)
                return Result<OrderProcessingEvent, string>.Failure("Order not found");
            
            var order = orderOpt.Value;
            
            var customerOpt = await _customerRepo.GetByIdAsync(order.CustomerId);
            if (customerOpt.IsNone)
                return Result<OrderProcessingEvent, string>.Failure("Customer not found");
            
            var customer = customerOpt.Value;
            
            // Convert to pure data structures
            var orderData = new OrderData(order.Id, order.CustomerId, order.Total);
            var customerData = new CustomerData(customer.Id, customer.Tier);
            
            // PURE: Business logic (no I/O, no side effects)
            var decision = OrderProcessingRules.ProcessOrder(orderData, customerData);
            
            // I/O: Handle decision based on type
            return await decision switch
            {
                OrderDecision.Accept accept => HandleAcceptedOrder(order, accept),
                OrderDecision.Reject reject => HandleRejectedOrder(order, reject),
                _ => throw new InvalidOperationException("Unknown decision type")
            };
        }
        
        private async Task<Result<OrderProcessingEvent, string>> HandleAcceptedOrder(
            Order order,
            OrderDecision.Accept decision)
        {
            // I/O: Update order
            order.SetFinalTotal(decision.FinalTotal);
            order.MarkAsProcessed();
            await _orderRepo.SaveAsync(order);
            
            // I/O: Publish event
            var evt = new OrderProcessingEvent.OrderProcessed(
                decision.OrderId,
                decision.FinalTotal);
            
            await _eventPublisher.PublishAsync(evt);
            
            return Result<OrderProcessingEvent, string>.Success(evt);
        }
        
        private async Task<Result<OrderProcessingEvent, string>> HandleRejectedOrder(
            Order order,
            OrderDecision.Reject decision)
        {
            // I/O: Update order
            order.MarkAsRejected(decision.Reason);
            await _orderRepo.SaveAsync(order);
            
            // I/O: Publish event
            var evt = new OrderProcessingEvent.OrderRejected(
                decision.OrderId,
                decision.Reason);
            
            await _eventPublisher.PublishAsync(evt);
            
            return Result<OrderProcessingEvent, string>.Success(evt);
        }
    }
}

// ============================================
// TESTING (Pure functions are trivial to test)
// ============================================
public class OrderProcessingRulesTests
{
    [Fact]
    public void ProcessOrder_BelowMinimum_RejectsOrder()
    {
        // Arrange: Pure data structures, no mocking
        var order = new OrderData(
            OrderId.New(),
            CustomerId.New(),
            Money.USD(5.00m));
        
        var customer = new CustomerData(
            CustomerId.New(),
            CustomerTier.Standard);
        
        // Act: Pure function call
        var decision = OrderProcessingRules.ProcessOrder(order, customer);
        
        // Assert: Simple verification
        Assert.IsType<OrderDecision.Reject>(decision);
        var reject = (OrderDecision.Reject)decision;
        Assert.Contains("below minimum", reject.Reason);
    }
    
    [Fact]
    public void ProcessOrder_VIPCustomer_AppliesDiscount()
    {
        // Arrange
        var order = new OrderData(
            OrderId.New(),
            CustomerId.New(),
            Money.USD(100.00m));
        
        var customer = new CustomerData(
            CustomerId.New(),
            CustomerTier.VIP);
        
        // Act
        var decision = OrderProcessingRules.ProcessOrder(order, customer);
        
        // Assert
        Assert.IsType<OrderDecision.Accept>(decision);
        var accept = (OrderDecision.Accept)decision;
        Assert.Equal(100.00m, accept.OriginalTotal.Amount);
        Assert.Equal(10.00m, accept.Discount.Amount);  // 10% of 100
        Assert.Equal(90.00m, accept.FinalTotal.Amount);
    }
    
    [Fact]
    public void ProcessOrder_StandardCustomer_NoDiscount()
    {
        // Arrange
        var order = new OrderData(
            OrderId.New(),
            CustomerId.New(),
            Money.USD(100.00m));
        
        var customer = new CustomerData(
            CustomerId.New(),
            CustomerTier.Standard);
        
        // Act
        var decision = OrderProcessingRules.ProcessOrder(order, customer);
        
        // Assert
        Assert.IsType<OrderDecision.Accept>(decision);
        var accept = (OrderDecision.Accept)decision;
        Assert.Equal(100.00m, accept.FinalTotal.Amount);  // No discount
        Assert.Equal(0m, accept.Discount.Amount);
    }
}
```

## Pattern Structure

```csharp
// ============================================
// FUNCTIONAL CORE: Pure business logic
// ============================================

// Input: Pure data (immutable records)
public sealed record InputData(/* ... */);

// Output: Pure decisions (discriminated union)
public abstract record Decision
{
    public sealed record Success(/* ... */) : Decision;
    public sealed record Failure(/* ... */) : Decision;
}

// Logic: Pure function (no I/O)
public static class BusinessRules
{
    public static Decision Process(InputData input)
    {
        // Pure calculations and business logic
        // No database, no I/O, no logging, no side effects
        // Just data in, decision out
    }
}

// ============================================
// IMPERATIVE SHELL: I/O and side effects
// ============================================

public class ApplicationService
{
    private readonly IRepository _repo;
    private readonly IEventPublisher _publisher;
    
    public async Task<Result> Handle(Command cmd)
    {
        // 1. I/O: Load data from external sources
        var data = await LoadData(cmd);
        
        // 2. PURE: Execute business logic (no I/O)
        var decision = BusinessRules.Process(data);
        
        // 3. I/O: Execute side effects based on decision
        await ExecuteSideEffects(decision);
        
        return Result.Success();
    }
}
```

## Advanced Example: Pricing Engine

```csharp
// ============================================
// FUNCTIONAL CORE
// ============================================
namespace Pricing.Domain.Core
{
    public sealed record PricingInput(
        ProductId ProductId,
        Money BasePrice,
        int Quantity,
        CustomerTier CustomerTier,
        Option<CouponCode> Coupon,
        DateTimeOffset PricingDate);
    
    public sealed record PricingResult(
        Money BasePrice,
        Money QuantityDiscount,
        Money TierDiscount,
        Money CouponDiscount,
        Money FinalPrice,
        IReadOnlyList<PricingExplanation> Explanations);
    
    public sealed record PricingExplanation(
        string Description,
        Money Amount);
    
    public static class PricingEngine
    {
        public static PricingResult CalculatePrice(PricingInput input)
        {
            var explanations = new List<PricingExplanation>();
            var currentPrice = input.BasePrice;
            
            explanations.Add(new PricingExplanation(
                "Base price",
                input.BasePrice));
            
            // Pure: Quantity discount
            var quantityDiscount = CalculateQuantityDiscount(
                currentPrice,
                input.Quantity);
            
            if (quantityDiscount.Amount > 0)
            {
                currentPrice = Money.FromAmount(
                    currentPrice.Amount - quantityDiscount.Amount,
                    currentPrice.Currency);
                
                explanations.Add(new PricingExplanation(
                    $"Quantity discount ({input.Quantity} units)",
                    Money.FromAmount(-quantityDiscount.Amount, currentPrice.Currency)));
            }
            
            // Pure: Customer tier discount
            var tierDiscount = CalculateTierDiscount(
                currentPrice,
                input.CustomerTier);
            
            if (tierDiscount.Amount > 0)
            {
                currentPrice = Money.FromAmount(
                    currentPrice.Amount - tierDiscount.Amount,
                    currentPrice.Currency);
                
                explanations.Add(new PricingExplanation(
                    $"Tier discount ({input.CustomerTier})",
                    Money.FromAmount(-tierDiscount.Amount, currentPrice.Currency)));
            }
            
            // Pure: Coupon discount
            var couponDiscount = input.Coupon.Match(
                onSome: coupon => CalculateCouponDiscount(currentPrice, coupon),
                onNone: () => Money.Zero);
            
            if (couponDiscount.Amount > 0)
            {
                currentPrice = Money.FromAmount(
                    currentPrice.Amount - couponDiscount.Amount,
                    currentPrice.Currency);
                
                explanations.Add(new PricingExplanation(
                    "Coupon discount",
                    Money.FromAmount(-couponDiscount.Amount, currentPrice.Currency)));
            }
            
            return new PricingResult(
                input.BasePrice,
                quantityDiscount,
                tierDiscount,
                couponDiscount,
                currentPrice,
                explanations);
        }
        
        private static Money CalculateQuantityDiscount(Money price, int quantity)
        {
            var discountRate = quantity switch
            {
                >= 100 => 0.15m,
                >= 50 => 0.10m,
                >= 10 => 0.05m,
                _ => 0m
            };
            
            return Money.FromAmount(price.Amount * discountRate, price.Currency);
        }
        
        private static Money CalculateTierDiscount(Money price, CustomerTier tier)
        {
            var discountRate = tier switch
            {
                CustomerTier.Platinum => 0.20m,
                CustomerTier.Gold => 0.15m,
                CustomerTier.Silver => 0.10m,
                CustomerTier.Bronze => 0.05m,
                _ => 0m
            };
            
            return Money.FromAmount(price.Amount * discountRate, price.Currency);
        }
        
        private static Money CalculateCouponDiscount(Money price, CouponCode coupon)
        {
            // Pure: Just calculate based on coupon properties
            return coupon.Type switch
            {
                CouponType.Percentage => Money.FromAmount(
                    price.Amount * coupon.Value / 100m,
                    price.Currency),
                CouponType.FixedAmount => Money.FromAmount(
                    Math.Min(coupon.Value, price.Amount),
                    price.Currency),
                _ => Money.Zero
            };
        }
    }
}
```

## Why It's a Problem

1. **Hard to test**: Business logic requires mocking infrastructure
2. **Hard to reason about**: Logic mixed with I/O effects
3. **Hard to reuse**: Pure logic can't be extracted
4. **Performance issues**: Can't optimize or cache pure calculations

## Symptoms

- Business logic methods marked `async` but do pure calculations
- Tests require extensive mocking of I/O
- Difficulty understanding what code does without running it
- Database or API calls interspersed with calculations
- Logging statements inside business logic

## Benefits

- **Testability**: Pure functions trivial to test
- **Reasoning**: Easy to understand input→output
- **Reusability**: Pure logic works anywhere
- **Performance**: Can cache, parallelize pure functions
- **Reliability**: No hidden I/O to fail

## See Also

- [Domain Model Isolation](./domain-model-isolation.md) — pure domain layer
- [Honest Functions](./honest-functions.md) — explicit about effects
- [Value Semantics](./value-semantics.md) — immutable data structures
- [Result Monad](./result-monad.md) — pure error handling
- [Discriminated Unions](./discriminated-unions.md) — modeling decisions
