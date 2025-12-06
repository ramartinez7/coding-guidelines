# Repository Pattern (Domain-Centric Data Access)

> Domain logic coupled to database—use repositories to abstract persistence and keep domain pure.

## Problem

Domain entities that directly interact with databases create tight coupling, make testing difficult, and leak infrastructure concerns into business logic.

## Example

### ❌ Before

```csharp
public class Order
{
    private readonly DbContext _db;
    
    public Order(DbContext db)
    {
        _db = db;  // Domain entity depends on infrastructure
    }
    
    public void AddItem(Product product, int quantity)
    {
        var item = new OrderItem { Product = product, Quantity = quantity };
        Items.Add(item);
        
        // Business logic mixed with persistence
        _db.OrderItems.Add(item);
        _db.SaveChanges();
    }
}
```

**Problems:**
- Domain logic coupled to database
- Hard to test without real database
- Persistence details leak into domain
- Cannot change storage strategy

### ✅ After

```csharp
/// <summary>
/// Repository abstracts persistence for aggregate roots.
/// </summary>
public interface IOrderRepository
{
    Task<Option<Order>> GetByIdAsync(OrderId id);
    Task<List<Order>> GetByCustomerAsync(CustomerId customerId);
    Task SaveAsync(Order order);
    Task DeleteAsync(OrderId id);
}

/// <summary>
/// Domain entity with no infrastructure dependencies.
/// </summary>
public sealed class Order
{
    // Pure domain logic, no database
    public void AddItem(ProductId productId, int quantity, Money price)
    {
        var item = new OrderItem(productId, quantity, price);
        _items.Add(item);
        RecalculateTotal();
    }
}

/// <summary>
/// EF Core implementation of repository.
/// </summary>
public class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    
    public async Task<Option<Order>> GetByIdAsync(OrderId id)
    {
        var order = await _db.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id);
        
        return order != null 
            ? Option<Order>.Some(order) 
            : Option<Order>.None;
    }
    
    public async Task SaveAsync(Order order)
    {
        var existing = await _db.Orders.FindAsync(order.Id);
        
        if (existing == null)
            _db.Orders.Add(order);
        else
            _db.Entry(existing).CurrentValues.SetValues(order);
        
        await _db.SaveChangesAsync();
    }
}

// Domain service uses repository
public class OrderService
{
    private readonly IOrderRepository _repository;
    
    public async Task ProcessOrder(OrderId orderId)
    {
        var orderResult = await _repository.GetByIdAsync(orderId);
        
        await orderResult.Match(
            onSome: async order =>
            {
                order.Process();  // Pure domain logic
                await _repository.SaveAsync(order);
            },
            onNone: () => Task.CompletedTask
        );
    }
}
```

## Specification Pattern Integration

```csharp
public interface IOrderRepository
{
    Task<List<Order>> FindAsync(ISpecification<Order> spec);
}

public class RecentOrdersSpecification : ISpecification<Order>
{
    private readonly DateTime _since;
    
    public RecentOrdersSpecification(DateTime since)
    {
        _since = since;
    }
    
    public Expression<Func<Order, bool>> ToExpression()
        => order => order.CreatedAt > _since;
}

// Usage
var spec = new RecentOrdersSpecification(DateTime.UtcNow.AddDays(-7));
var recentOrders = await _repository.FindAsync(spec);
```

## Unit of Work Pattern

```csharp
public interface IUnitOfWork : IDisposable
{
    IOrderRepository Orders { get; }
    ICustomerRepository Customers { get; }
    IProductRepository Products { get; }
    
    Task<int> SaveChangesAsync();
}

// Ensures all changes saved together
public class OrderService
{
    private readonly IUnitOfWork _uow;
    
    public async Task TransferOrder(OrderId orderId, CustomerId newCustomerId)
    {
        var order = (await _uow.Orders.GetByIdAsync(orderId)).Value;
        var customer = (await _uow.Customers.GetByIdAsync(newCustomerId)).Value;
        
        order.TransferTo(newCustomerId);
        customer.RecordOrderTransfer(orderId);
        
        // Both changes saved in single transaction
        await _uow.SaveChangesAsync();
    }
}
```

## Testing with In-Memory Repository

```csharp
public class InMemoryOrderRepository : IOrderRepository
{
    private readonly Dictionary<OrderId, Order> _orders = new();
    
    public Task<Option<Order>> GetByIdAsync(OrderId id)
    {
        return Task.FromResult(
            _orders.TryGetValue(id, out var order)
                ? Option<Order>.Some(order)
                : Option<Order>.None);
    }
    
    public Task SaveAsync(Order order)
    {
        _orders[order.Id] = order;
        return Task.CompletedTask;
    }
}

// Testing without database
[Fact]
public async Task ProcessOrder_ValidOrder_Succeeds()
{
    var repository = new InMemoryOrderRepository();
    var service = new OrderService(repository);
    
    var order = Order.Create(OrderId.New(), customerId);
    await repository.SaveAsync(order);
    
    await service.ProcessOrder(order.Id);
    
    var processed = (await repository.GetByIdAsync(order.Id)).Value;
    Assert.Equal(OrderStatus.Processed, processed.Status);
}
```

## Why It's a Problem

1. **Tight coupling**: Domain depends on infrastructure
2. **Hard to test**: Requires real database
3. **Limited flexibility**: Can't swap persistence
4. **Leaky abstraction**: SQL queries in domain

## Benefits

- **Separation of concerns**: Domain isolated from persistence
- **Testability**: Easy to mock or use in-memory
- **Flexibility**: Can change storage strategy
- **Clean domain**: Pure business logic

## See Also

- [Aggregate Roots](./aggregate-roots.md) — consistency boundaries
- [Specification Pattern](./specification-pattern.md) — query encapsulation
- [Domain Events](./domain-events.md) — decoupling side effects
