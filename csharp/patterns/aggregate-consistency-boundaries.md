# Aggregate Consistency Boundaries (Transactional Consistency Through Design)

> Entities modified in multiple transactions lead to inconsistent state—define aggregate boundaries to ensure atomic, consistent updates across related objects.

## Problem

Without clear boundaries, multiple entities can be modified independently, leading to inconsistent state. Transactions span too many entities, causing performance problems and concurrency conflicts.

## Example

### ❌ Before

```csharp
// Entities without clear boundaries
public class Order
{
    public OrderId Id { get; set; }
    public CustomerId CustomerId { get; set; }
    public List<OrderLine> Lines { get; set; }
    public decimal Total { get; set; }
}

public class OrderLine
{
    public OrderLineId Id { get; set; }
    public OrderId OrderId { get; set; }
    public ProductId ProductId { get; set; }
    public int Quantity { get; set; }
}

public class Inventory
{
    public ProductId ProductId { get; set; }
    public int Available { get; set; }
}

// Service modifying multiple entities across aggregates
public class OrderService
{
    public async Task CreateOrder(CreateOrderCommand cmd)
    {
        // Transaction spans multiple aggregates
        using var transaction = await _db.BeginTransactionAsync();
        
        try
        {
            var order = new Order { CustomerId = cmd.CustomerId };
            
            foreach (var item in cmd.Items)
            {
                // Direct access to inventory
                var inventory = await _db.Inventory
                    .FirstAsync(i => i.ProductId == item.ProductId);
                
                if (inventory.Available < item.Quantity)
                    throw new InvalidOperationException("Insufficient inventory");
                
                inventory.Available -= item.Quantity;  // Modifying another aggregate!
                
                order.Lines.Add(new OrderLine
                {
                    ProductId = item.ProductId,
                    Quantity = item.Quantity
                });
            }
            
            await _db.Orders.AddAsync(order);
            await _db.SaveChangesAsync();
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

**Problems:**
- Transaction spans Order and Inventory aggregates
- Inventory modified from outside its aggregate
- High contention on inventory table
- Difficult to maintain consistency
- Can't scale horizontally

### ✅ After

```csharp
// ============================================
// AGGREGATE 1: Order (with internal consistency)
// ============================================
public sealed class Order
{
    private readonly List<OrderLine> _lines = new();
    
    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public IReadOnlyList<OrderLine> Items => _lines.AsReadOnly();
    public Money Total { get; private set; }
    public OrderStatus Status { get; private set; }
    
    private Order(OrderId id, CustomerId customerId)
    {
        Id = id;
        CustomerId = customerId;
        Total = Money.Zero;
        Status = OrderStatus.Pending;
    }
    
    public static Order Create(OrderId id, CustomerId customerId) =>
        new Order(id, customerId);
    
    public Result<Unit, string> AddLine(ProductId productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Pending)
            return Result<Unit, string>.Failure("Cannot modify confirmed order");
        
        if (quantity <= 0)
            return Result<Unit, string>.Failure("Quantity must be positive");
        
        var line = new OrderLine(productId, quantity, unitPrice);
        _lines.Add(line);
        
        RecalculateTotal();
        
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public Result<Unit, string> Confirm()
    {
        if (Status != OrderStatus.Pending)
            return Result<Unit, string>.Failure("Order already confirmed");
        
        if (_lines.Count == 0)
            return Result<Unit, string>.Failure("Cannot confirm empty order");
        
        Status = OrderStatus.Confirmed;
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    private void RecalculateTotal()
    {
        Total = _lines
            .Select(line => line.LineTotal)
            .Aggregate(Money.Zero, (sum, total) => sum + total);
    }
}

public sealed record OrderLine(ProductId ProductId, int Quantity, Money UnitPrice)
{
    public Money LineTotal => UnitPrice * Quantity;
}

// ============================================
// AGGREGATE 2: Inventory (separate consistency boundary)
// ============================================
public sealed class InventoryItem
{
    public ProductId ProductId { get; }
    public int Available { get; private set; }
    public int Reserved { get; private set; }
    private readonly List<Reservation> _reservations = new();
    
    public int TotalOnHand => Available + Reserved;
    
    private InventoryItem(ProductId productId, int available)
    {
        ProductId = productId;
        Available = available;
        Reserved = 0;
    }
    
    public static InventoryItem Create(ProductId productId, int initialQuantity)
    {
        if (initialQuantity < 0)
            throw new ArgumentException("Initial quantity cannot be negative");
        
        return new InventoryItem(productId, initialQuantity);
    }
    
    public Result<ReservationId, string> Reserve(int quantity, OrderId orderId)
    {
        if (quantity <= 0)
            return Result<ReservationId, string>.Failure("Quantity must be positive");
        
        if (Available < quantity)
            return Result<ReservationId, string>.Failure(
                $"Insufficient inventory. Available: {Available}, Requested: {quantity}");
        
        var reservationId = ReservationId.New();
        var reservation = new Reservation(reservationId, orderId, quantity);
        
        Available -= quantity;
        Reserved += quantity;
        _reservations.Add(reservation);
        
        return Result<ReservationId, string>.Success(reservationId);
    }
    
    public Result<Unit, string> Commit(ReservationId reservationId)
    {
        var reservation = _reservations.FirstOrDefault(r => r.Id == reservationId);
        if (reservation == null)
            return Result<Unit, string>.Failure("Reservation not found");
        
        Reserved -= reservation.Quantity;
        _reservations.Remove(reservation);
        
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public Result<Unit, string> Rollback(ReservationId reservationId)
    {
        var reservation = _reservations.FirstOrDefault(r => r.Id == reservationId);
        if (reservation == null)
            return Result<Unit, string>.Failure("Reservation not found");
        
        Available += reservation.Quantity;
        Reserved -= reservation.Quantity;
        _reservations.Remove(reservation);
        
        return Result<Unit, string>.Success(Unit.Value);
    }
}

public sealed record Reservation(
    ReservationId Id,
    OrderId OrderId,
    int Quantity);

// ============================================
// DOMAIN SERVICE: Coordinates across aggregates
// ============================================
public sealed class OrderCreationService
{
    private readonly IOrderRepository _orderRepo;
    private readonly IInventoryRepository _inventoryRepo;
    
    public OrderCreationService(
        IOrderRepository orderRepo,
        IInventoryRepository inventoryRepo)
    {
        _orderRepo = orderRepo;
        _inventoryRepo = inventoryRepo;
    }
    
    public async Task<Result<Order, string>> CreateOrder(CreateOrderCommand cmd)
    {
        var order = Order.Create(OrderId.New(), cmd.CustomerId);
        var reservations = new List<(ProductId, ReservationId)>();
        
        try
        {
            // Phase 1: Reserve inventory (separate aggregate transactions)
            foreach (var item in cmd.Items)
            {
                var inventoryOpt = await _inventoryRepo.GetByProductIdAsync(item.ProductId);
                
                if (inventoryOpt.IsNone)
                    return Result<Order, string>.Failure($"Product {item.ProductId} not found");
                
                var inventory = inventoryOpt.Value;
                
                var reserveResult = inventory.Reserve(item.Quantity, order.Id);
                if (!reserveResult.IsSuccess)
                {
                    // Rollback previous reservations
                    await RollbackReservations(reservations);
                    return Result<Order, string>.Failure(reserveResult.Error);
                }
                
                reservations.Add((item.ProductId, reserveResult.Value));
                
                // Save inventory (separate transaction per aggregate)
                await _inventoryRepo.SaveAsync(inventory);
            }
            
            // Phase 2: Create order (separate aggregate transaction)
            foreach (var item in cmd.Items)
            {
                var addLineResult = order.AddLine(item.ProductId, item.Quantity, item.UnitPrice);
                if (!addLineResult.IsSuccess)
                {
                    await RollbackReservations(reservations);
                    return Result<Order, string>.Failure(addLineResult.Error);
                }
            }
            
            var confirmResult = order.Confirm();
            if (!confirmResult.IsSuccess)
            {
                await RollbackReservations(reservations);
                return Result<Order, string>.Failure(confirmResult.Error);
            }
            
            // Save order (separate transaction)
            await _orderRepo.SaveAsync(order);
            
            // Phase 3: Commit reservations (separate aggregate transactions)
            foreach (var (productId, reservationId) in reservations)
            {
                var inventoryOpt = await _inventoryRepo.GetByProductIdAsync(productId);
                if (inventoryOpt.IsSome)
                {
                    var inventory = inventoryOpt.Value;
                    inventory.Commit(reservationId);
                    await _inventoryRepo.SaveAsync(inventory);
                }
            }
            
            return Result<Order, string>.Success(order);
        }
        catch
        {
            await RollbackReservations(reservations);
            throw;
        }
    }
    
    private async Task RollbackReservations(
        List<(ProductId ProductId, ReservationId ReservationId)> reservations)
    {
        foreach (var (productId, reservationId) in reservations)
        {
            var inventoryOpt = await _inventoryRepo.GetByProductIdAsync(productId);
            if (inventoryOpt.IsSome)
            {
                var inventory = inventoryOpt.Value;
                inventory.Rollback(reservationId);
                await _inventoryRepo.SaveAsync(inventory);
            }
        }
    }
}
```

## Aggregate Boundary Rules

```csharp
/// <summary>
/// Rule 1: Aggregate boundaries are consistency boundaries.
/// Modifications within aggregate are atomic.
/// </summary>
public sealed class ShoppingCart
{
    // All modifications to cart and items happen atomically
    private readonly List<CartItem> _items = new();
    
    public Result<Unit, string> AddItem(ProductId productId, int quantity)
    {
        // Internal consistency maintained
        var existing = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existing != null)
        {
            existing.IncreaseQuantity(quantity);
        }
        else
        {
            _items.Add(new CartItem(productId, quantity));
        }
        
        return Result<Unit, string>.Success(Unit.Value);
    }
}

/// <summary>
/// Rule 2: Reference other aggregates by ID only, not by object reference.
/// </summary>
public sealed class Order
{
    public CustomerId CustomerId { get; }  // Reference by ID
    
    // ❌ NOT: public Customer Customer { get; }  // Don't embed other aggregate
}

public sealed class Customer
{
    public CustomerId Id { get; }
    public PersonName Name { get; }
    
    // ❌ NOT: public List<Order> Orders { get; }  // Don't embed child aggregates
}

/// <summary>
/// Rule 3: One transaction = One aggregate instance.
/// </summary>
public interface IOrderRepository
{
    // Each save operation is one aggregate, one transaction
    Task SaveAsync(Order order);
    
    // ❌ NOT: Task SaveMultipleAsync(List<Order> orders);
}

/// <summary>
/// Rule 4: Use eventual consistency between aggregates.
/// </summary>
public sealed class OrderConfirmedHandler : IEventHandler<OrderConfirmedEvent>
{
    private readonly IInventoryRepository _inventoryRepo;
    
    public async Task HandleAsync(OrderConfirmedEvent evt)
    {
        // Eventually consistent: inventory updated after order confirmed
        foreach (var item in evt.Items)
        {
            var inventoryOpt = await _inventoryRepo.GetByProductIdAsync(item.ProductId);
            if (inventoryOpt.IsSome)
            {
                var inventory = inventoryOpt.Value;
                inventory.DecrementStock(item.Quantity);
                await _inventoryRepo.SaveAsync(inventory);
            }
        }
    }
}
```

## Determining Aggregate Boundaries

```csharp
// Example: E-commerce domain

// ✅ GOOD: Small aggregate, clear boundary
public sealed class Product
{
    public ProductId Id { get; }
    public ProductName Name { get; }
    public Money Price { get; private set; }
    public ProductCategory Category { get; }
    
    // Only product data, no inventory or orders
}

// ✅ GOOD: Inventory is separate aggregate
public sealed class InventoryItem
{
    public ProductId ProductId { get; }  // Reference to Product by ID
    public int Quantity { get; private set; }
    
    // Inventory concerns separate from product
}

// ❌ BAD: Too large, mixed concerns
public class ProductAggregate
{
    public ProductId Id { get; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    
    // Don't include inventory in product aggregate
    public int QuantityOnHand { get; set; }
    public int QuantityReserved { get; set; }
    
    // Don't include reviews in product aggregate
    public List<Review> Reviews { get; set; }
    
    // Don't include orders in product aggregate
    public List<Order> Orders { get; set; }
}

// ✅ GOOD: Reviews are separate aggregate
public sealed class ProductReview
{
    public ReviewId Id { get; }
    public ProductId ProductId { get; }  // Reference by ID
    public CustomerId CustomerId { get; }  // Reference by ID
    public Rating Rating { get; }
    public ReviewText Text { get; }
}
```

## Why It's a Problem

1. **Inconsistent state**: No clear transaction boundaries
2. **Performance**: Large transactions lock many rows
3. **Concurrency conflicts**: Multiple users conflict on same entities
4. **Scaling issues**: Can't distribute aggregates across services

## Symptoms

- Transactions spanning many tables
- Frequent deadlocks and timeouts
- Difficult to reason about consistency
- Performance degradation under load

## Benefits

- **Clear consistency boundaries**: Know what's atomic
- **Better performance**: Smaller transactions, less contention
- **Scalability**: Aggregates can be distributed
- **Maintainability**: Clear boundaries between concepts

## See Also

- [Aggregate Roots](./aggregate-roots.md) — enforcing invariants
- [Repository Pattern](./repository-pattern.md) — persistence per aggregate
- [Domain Events](./domain-events.md) — eventual consistency
- [Bounded Contexts](./bounded-contexts.md) — strategic boundaries
- [Domain Services](./domain-services.md) — coordinating across aggregates
