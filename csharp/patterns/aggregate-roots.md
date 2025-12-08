# Aggregate Roots (Enforcing Invariants Through Boundaries)

> Entities with direct public setters allow invariants to be broken—use aggregate roots to enforce consistency boundaries.

## Problem

When domain objects expose public setters or allow direct manipulation of child entities, business rules can be bypassed. There's no single place to enforce invariants that span multiple entities.

## Example

### ❌ Before

```csharp
public class Order
{
    public OrderId Id { get; set; }
    public CustomerId CustomerId { get; set; }
    public List<OrderItem> Items { get; set; } = new();
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
}

public class OrderItem
{
    public ProductId ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal Price { get; set; }
}

// Invariants can be broken
var order = new Order { CustomerId = customerId };
order.Items.Add(new OrderItem 
{ 
    ProductId = productId, 
    Quantity = -5,  // Invalid!
    Price = 100
});
order.Total = 42;  // Wrong! Doesn't match items
order.Status = OrderStatus.Shipped;  // Can skip validation
```

**Problems:**
- Public setters allow breaking invariants
- Total can be inconsistent with items
- Status can be changed without validation
- No enforcement of business rules

### ✅ After

```csharp
/// <summary>
/// Aggregate root: enforces consistency boundary for Order and its items.
/// </summary>
public sealed class Order
{
    private readonly List<OrderItem> _items = new();
    
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    public Money Total { get; private set; }
    public OrderStatus Status { get; private set; }
    
    // Private constructor—can only create through factory
    private Order(OrderId id, CustomerId customerId)
    {
        Id = id;
        CustomerId = customerId;
        Total = Money.Zero;
        Status = OrderStatus.Draft;
    }
    
    public static Order Create(OrderId id, CustomerId customerId)
    {
        return new Order(id, customerId);
    }
    
    // Business logic enforced in methods
    public Result<Unit, string> AddItem(ProductId productId, int quantity, Money price)
    {
        if (Status != OrderStatus.Draft)
            return Result<Unit, string>.Failure("Cannot modify submitted order");
        
        if (quantity <= 0)
            return Result<Unit, string>.Failure("Quantity must be positive");
        
        if (price.Amount <= 0)
            return Result<Unit, string>.Failure("Price must be positive");
        
        var item = new OrderItem(productId, quantity, price);
        _items.Add(item);
        
        // Invariant: Total always matches sum of items
        RecalculateTotal();
        
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public Result<Unit, string> RemoveItem(ProductId productId)
    {
        if (Status != OrderStatus.Draft)
            return Result<Unit, string>.Failure("Cannot modify submitted order");
        
        var item = _items.FirstOrDefault(i => i.ProductId == productId);
        if (item == null)
            return Result<Unit, string>.Failure("Item not found");
        
        _items.Remove(item);
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
    
    public Result<Unit, string> Ship()
    {
        if (Status != OrderStatus.Submitted)
            return Result<Unit, string>.Failure("Can only ship submitted orders");
        
        Status = OrderStatus.Shipped;
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    private void RecalculateTotal()
    {
        Total = _items
            .Select(i => i.Price * i.Quantity)
            .Aggregate(Money.Zero, (sum, price) => sum + price);
    }
}

/// <summary>
/// Value object within Order aggregate.
/// Cannot be modified directly—only through aggregate root.
/// </summary>
public sealed class OrderItem
{
    public ProductId ProductId { get; }
    public int Quantity { get; }
    public Money Price { get; }
    
    internal OrderItem(ProductId productId, int quantity, Money price)
    {
        ProductId = productId;
        Quantity = quantity;
        Price = price;
    }
}

// Usage: Invariants enforced
var order = Order.Create(orderId, customerId);
order.AddItem(productId, 5, Money.USD(100));  // ✓ Valid
order.AddItem(productId2, -5, Money.USD(100));  // ✗ Returns error
order.Submit();  // Changes status with validation
```

## Repository Pattern for Aggregates

```csharp
/// <summary>
/// Repository operates only on aggregate roots.
/// </summary>
public interface IOrderRepository
{
    Task<Option<Order>> GetByIdAsync(OrderId id);
    Task SaveAsync(Order order);
    Task DeleteAsync(OrderId id);
    
    // ❌ No methods to access OrderItems directly
    // ❌ No methods to modify Order without going through aggregate
}

public class OrderRepository : IOrderRepository
{
    public async Task SaveAsync(Order order)
    {
        // Save entire aggregate in one transaction
        using var transaction = await _db.BeginTransactionAsync();
        
        try
        {
            await SaveOrderAsync(order);
            await SaveOrderItemsAsync(order.Items);
            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

## Multiple Aggregates

```csharp
// Different aggregates reference each other by ID only
public sealed class Order
{
    public CustomerId CustomerId { get; }  // Reference by ID
    
    // ❌ NOT: public Customer Customer { get; }  // Don't embed other aggregates
}

public sealed class Customer
{
    public CustomerId Id { get; }
    public Email Email { get; }
    
    // ❌ NOT: public List<Order> Orders { get; }  // Don't embed child aggregates
}

// Service coordinates across aggregates
public class OrderService
{
    public async Task<Result<Order, string>> CreateOrder(
        CustomerId customerId,
        List<(ProductId, int, Money)> items)
    {
        // Load customer aggregate
        var customerResult = await _customerRepo.GetByIdAsync(customerId);
        
        return await customerResult.Match(
            onSome: async customer =>
            {
                // Validate across aggregates
                if (!customer.IsActive)
                    return Result<Order, string>.Failure("Customer not active");
                
                // Create order aggregate
                var order = Order.Create(OrderId.New(), customerId);
                foreach (var (productId, quantity, price) in items)
                {
                    var result = order.AddItem(productId, quantity, price);
                    if (!result.IsSuccess)
                        return Result<Order, string>.Failure(result.Error);
                }
                
                await _orderRepo.SaveAsync(order);
                return Result<Order, string>.Success(order);
            },
            onNone: () => Task.FromResult(
                Result<Order, string>.Failure("Customer not found"))
        );
    }
}
```

## Why It's a Problem

1. **Broken invariants**: Direct access bypasses business rules
2. **Inconsistent state**: Total doesn't match items
3. **Scattered validation**: Rules duplicated across codebase
4. **No transactional boundary**: Partial updates possible

## Symptoms

- Public setters on entities
- Business rules in multiple places
- Inconsistent data in database
- `List<T>` instead of `IReadOnlyList<T>`

## Benefits

- **Consistency enforced**: Invariants guaranteed by aggregate
- **Single source of truth**: Business rules in one place
- **Transactional boundary**: Save entire aggregate atomically
- **Encapsulation**: Internal state hidden

## See Also

- [Value Semantics](./value-semantics.md) — immutable value objects
- [Enforcing Call Order](./enforcing-call-order.md) — state transitions
- [Domain Events](./domain-events.md) — announcing changes
- [Repository Pattern](./repository-pattern.md) — persistence
- [Domain Invariants](./domain-invariants.md) — enforcing business rules at construction
- [Type-Safe Workflow Modeling](./type-safe-workflow-modeling.md) — modeling aggregate state transitions
