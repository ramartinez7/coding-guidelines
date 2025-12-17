# LINQ Query Optimization

> Write efficient LINQ queries by understanding deferred execution, avoiding multiple enumeration, and using appropriate query methods.

## Problem

Inefficient LINQ queries cause unnecessary database roundtrips, multiple enumerations, and N+1 query problems, leading to poor performance.

## Example

### ❌ Before

```csharp
public class OrderService
{
    // N+1 query problem
    public List<OrderDto> GetOrders()
    {
        var orders = _context.Orders.ToList();

        return orders.Select(o => new OrderDto
        {
            OrderId = o.Id,
            CustomerName = _context.Customers
                .First(c => c.Id == o.CustomerId).Name,  // Query per order!
            ProductNames = _context.Products
                .Where(p => o.ProductIds.Contains(p.Id))
                .Select(p => p.Name)
                .ToList()  // Query per order!
        }).ToList();
    }

    // Multiple enumeration
    public OrderStatistics GetStatistics(List<Order> orders)
    {
        var filtered = orders.Where(o => o.Status == OrderStatus.Completed);

        return new OrderStatistics
        {
            Count = filtered.Count(),  // Enumeration 1
            Total = filtered.Sum(o => o.Total),  // Enumeration 2
            Average = filtered.Average(o => o.Total)  // Enumeration 3
        };
    }

    // Loading everything into memory
    public List<Order> GetExpensiveOrders()
    {
        var allOrders = _context.Orders.ToList();  // Loads everything!
        return allOrders.Where(o => o.Total > 1000).ToList();
    }
}
```

### ✅ After

```csharp
public class OrderService
{
    // ✅ Eager loading to avoid N+1
    public List<OrderDto> GetOrders()
    {
        return _context.Orders
            .Include(o => o.Customer)
            .Include(o => o.Products)
            .Select(o => new OrderDto
            {
                OrderId = o.Id,
                CustomerName = o.Customer.Name,
                ProductNames = o.Products.Select(p => p.Name).ToList()
            })
            .ToList();
    }

    // ✅ Single enumeration with materialization
    public OrderStatistics GetStatistics(List<Order> orders)
    {
        var filtered = orders
            .Where(o => o.Status == OrderStatus.Completed)
            .ToList();  // Materialize once

        return new OrderStatistics
        {
            Count = filtered.Count,  // Property, not method
            Total = filtered.Sum(o => o.Total),
            Average = filtered.Average(o => o.Total)
        };
    }

    // ✅ Server-side filtering
    public List<Order> GetExpensiveOrders()
    {
        return _context.Orders
            .Where(o => o.Total > 1000)  // Filter on database
            .ToList();  // Only retrieve matching records
    }
}
```

## Optimization Techniques

### 1. Avoid N+1 Query Problems

```csharp
// ❌ N+1 queries
public List<OrderWithCustomer> GetOrdersWithCustomers()
{
    var orders = _context.Orders.ToList();

    return orders.Select(o => new OrderWithCustomer
    {
        Order = o,
        Customer = _context.Customers.Find(o.CustomerId)  // N queries!
    }).ToList();
}

// ✅ Use Include for eager loading
public List<OrderWithCustomer> GetOrdersWithCustomers()
{
    return _context.Orders
        .Include(o => o.Customer)  // Single query with JOIN
        .Select(o => new OrderWithCustomer
        {
            Order = o,
            Customer = o.Customer
        })
        .ToList();
}
```

### 2. Avoid Multiple Enumeration

```csharp
// ❌ Enumerating IEnumerable multiple times
public void ProcessOrders(IEnumerable<Order> orders)
{
    if (!orders.Any()) return;  // Enumeration 1
    
    var total = orders.Sum(o => o.Total);  // Enumeration 2
    var count = orders.Count();  // Enumeration 3
}

// ✅ Materialize once if needed multiple times
public void ProcessOrders(IEnumerable<Order> orders)
{
    var orderList = orders.ToList();  // Single enumeration

    if (orderList.Count == 0) return;
    
    var total = orderList.Sum(o => o.Total);
    var count = orderList.Count;
}
```

### 3. Filter Before Projection

```csharp
// ❌ Project then filter (processes unnecessary data)
public List<string> GetActiveCustomerNames()
{
    return _context.Customers
        .Select(c => new { c.Name, c.IsActive })  // Select all
        .Where(x => x.IsActive)  // Filter after projection
        .Select(x => x.Name)
        .ToList();
}

// ✅ Filter then project (more efficient)
public List<string> GetActiveCustomerNames()
{
    return _context.Customers
        .Where(c => c.IsActive)  // Filter first
        .Select(c => c.Name)  // Then project
        .ToList();
}
```

### 4. Use Any() Instead of Count() for Existence Checks

```csharp
// ❌ Count() enumerates entire sequence
if (_context.Orders.Count() > 0)
{
    ProcessOrders();
}

if (_context.Orders.Count(o => o.Status == OrderStatus.Pending) > 0)
{
    ProcessPendingOrders();
}

// ✅ Any() stops at first match
if (_context.Orders.Any())
{
    ProcessOrders();
}

if (_context.Orders.Any(o => o.Status == OrderStatus.Pending))
{
    ProcessPendingOrders();
}
```

### 5. Use FirstOrDefault Instead of Where().FirstOrDefault()

```csharp
// ❌ Where then First (less efficient SQL)
var order = _context.Orders
    .Where(o => o.Id == orderId)
    .FirstOrDefault();

// ✅ FirstOrDefault with predicate (better SQL)
var order = _context.Orders
    .FirstOrDefault(o => o.Id == orderId);
```

### 6. Avoid ToList() in the Middle of Query Chain

```csharp
// ❌ Materializes too early, loses query composition
public List<Order> GetRecentExpensiveOrders()
{
    var allOrders = _context.Orders.ToList();  // Loads everything!

    return allOrders
        .Where(o => o.CreatedAt > DateTime.UtcNow.AddDays(-30))
        .Where(o => o.Total > 1000)
        .OrderByDescending(o => o.CreatedAt)
        .ToList();
}

// ✅ Compose query, materialize at the end
public List<Order> GetRecentExpensiveOrders()
{
    return _context.Orders
        .Where(o => o.CreatedAt > DateTime.UtcNow.AddDays(-30))
        .Where(o => o.Total > 1000)
        .OrderByDescending(o => o.CreatedAt)
        .ToList();  // Single database query
}
```

### 7. Use AsNoTracking for Read-Only Queries

```csharp
// ❌ Default tracking (overhead for read-only data)
public List<OrderDto> GetOrdersForDisplay()
{
    return _context.Orders
        .Select(o => new OrderDto { /* ... */ })
        .ToList();  // EF tracks entities unnecessarily
}

// ✅ Disable tracking for read-only operations
public List<OrderDto> GetOrdersForDisplay()
{
    return _context.Orders
        .AsNoTracking()  // Better performance
        .Select(o => new OrderDto { /* ... */ })
        .ToList();
}
```

### 8. Use Pagination for Large Result Sets

```csharp
// ❌ Loading all records
public List<Order> GetOrders()
{
    return _context.Orders.ToList();  // Could be millions!
}

// ✅ Use Skip and Take for pagination
public List<Order> GetOrders(int pageNumber, int pageSize)
{
    return _context.Orders
        .OrderBy(o => o.CreatedAt)  // Must order for stable pagination
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .ToList();
}
```

### 9. Avoid Unnecessary Ordering

```csharp
// ❌ Ordering when not needed
public int GetOrderCount()
{
    return _context.Orders
        .OrderBy(o => o.CreatedAt)  // Unnecessary sort!
        .Count();
}

// ✅ Remove unnecessary ordering
public int GetOrderCount()
{
    return _context.Orders.Count();
}
```

### 10. Use Compiled Queries for Repeated Queries

```csharp
// ❌ Query compiled every time
public Order GetOrder(OrderId orderId)
{
    return _context.Orders
        .Include(o => o.Customer)
        .FirstOrDefault(o => o.Id == orderId);
}

// ✅ Compile query once, reuse
private static readonly Func<OrderDbContext, OrderId, Order> GetOrderQuery =
    EF.CompileQuery((OrderDbContext context, OrderId orderId) =>
        context.Orders
            .Include(o => o.Customer)
            .FirstOrDefault(o => o.Id == orderId));

public Order GetOrder(OrderId orderId)
{
    return GetOrderQuery(_context, orderId);
}
```

### 11. Avoid Client-Side Evaluation

```csharp
// ❌ Client-side method execution
public List<Order> GetRecentOrders()
{
    return _context.Orders
        .Where(o => IsRecent(o.CreatedAt))  // Can't translate to SQL!
        .ToList();
}

private bool IsRecent(DateTime date)
{
    return date > DateTime.UtcNow.AddDays(-7);
}

// ✅ Use expressions that translate to SQL
public List<Order> GetRecentOrders()
{
    var cutoffDate = DateTime.UtcNow.AddDays(-7);

    return _context.Orders
        .Where(o => o.CreatedAt > cutoffDate)  // Translates to SQL
        .ToList();
}
```

### 12. Use Select to Project Early

```csharp
// ❌ Loading entire entities when only need specific fields
public List<string> GetCustomerNames()
{
    var customers = _context.Customers.ToList();  // Loads all columns!
    return customers.Select(c => c.Name).ToList();
}

// ✅ Project in database
public List<string> GetCustomerNames()
{
    return _context.Customers
        .Select(c => c.Name)  // Only retrieves Name column
        .ToList();
}
```

## Symptoms

- Slow query performance
- High database CPU usage
- Many small database queries instead of few large ones
- OutOfMemoryException when loading large datasets
- Timeout errors on database queries

## Benefits

- **Better performance** with optimized database queries
- **Reduced memory usage** by avoiding loading unnecessary data
- **Fewer database roundtrips** with proper eager loading
- **Faster response times** for users
- **Lower infrastructure costs** with efficient queries

## See Also

- [First-Class Collections](./first-class-collections.md) — Wrapping collections with domain operations
- [Repository Pattern](./repository-pattern.md) — Abstracting data access
- [Primitive Collections](./primitive-collections.md) — Avoiding primitive collection types
