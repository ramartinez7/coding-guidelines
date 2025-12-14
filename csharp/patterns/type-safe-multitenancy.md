# Type-Safe Multitenancy (Tenant Isolation)

> Tenant IDs passed as primitives allow cross-tenant data leaks—use typed tenant contexts to enforce tenant isolation at compile time.

## Problem

Multi-tenant applications that pass tenant identifiers as strings or integers throughout the codebase risk data leakage when tenant context is forgotten, passed incorrectly, or bypassed. The compiler can't verify that operations are tenant-scoped.

## Example

### ❌ Before

```csharp
public class OrderRepository
{
    public async Task<Order> GetOrderAsync(string tenantId, Guid orderId)
    {
        // Easy to forget tenant filter
        return await _db.Orders
            .Where(o => o.Id == orderId)  // Missing tenant check!
            .FirstOrDefaultAsync();
    }
    
    public async Task<List<Order>> GetAllOrdersAsync(string tenantId)
    {
        return await _db.Orders
            .Where(o => o.TenantId == tenantId)
            .ToListAsync();
    }
}
```

### ✅ After

```csharp
public readonly record struct TenantId(Guid Value);

public sealed class TenantContext
{
    public TenantId Id { get; }
    
    private TenantContext(TenantId id)
    {
        Id = id;
    }
    
    public static Result<TenantContext, string> Create(Guid tenantId)
    {
        if (tenantId == Guid.Empty)
            return Result<TenantContext, string>.Failure("Invalid tenant ID");
        
        return Result<TenantContext, string>.Success(
            new TenantContext(new TenantId(tenantId)));
    }
}

public sealed class TenantScoped<T>
{
    public TenantContext Tenant { get; }
    public T Value { get; }
    
    public TenantScoped(TenantContext tenant, T value)
    {
        Tenant = tenant;
        Value = value;
    }
    
    public TenantScoped<TNext> Map<TNext>(Func<T, TNext> mapper)
    {
        return new TenantScoped<TNext>(Tenant, mapper(Value));
    }
}

public interface ITenantRepository<T>
{
    Task<Option<TenantScoped<T>>> GetByIdAsync(TenantContext tenant, Guid id);
    Task<IReadOnlyList<TenantScoped<T>>> GetAllAsync(TenantContext tenant);
}

public class OrderRepository : ITenantRepository<Order>
{
    public async Task<Option<TenantScoped<Order>>> GetByIdAsync(
        TenantContext tenant,
        Guid orderId)
    {
        var order = await _db.Orders
            .Where(o => o.TenantId == tenant.Id.Value)  // Forced to filter
            .Where(o => o.Id == orderId)
            .FirstOrDefaultAsync();
        
        return order is null
            ? Option<TenantScoped<Order>>.None()
            : Option<TenantScoped<Order>>.Some(
                new TenantScoped<Order>(tenant, order));
    }
    
    public async Task<IReadOnlyList<TenantScoped<Order>>> GetAllAsync(
        TenantContext tenant)
    {
        var orders = await _db.Orders
            .Where(o => o.TenantId == tenant.Id.Value)
            .ToListAsync();
        
        return orders
            .Select(o => new TenantScoped<Order>(tenant, o))
            .ToList();
    }
}
```

## See Also

- [Authentication Context](./authentication-context.md)
- [Capability Security](./capability-security.md)
- [Strongly Typed IDs](./strongly-typed-ids.md)
