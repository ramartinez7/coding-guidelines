# Type-Safe Messaging (Compile-Time Message Contract Safety)

> Message contracts as strings or dynamic objects fail at runtime—use typed messages with serialization contracts to catch integration errors early.

## Problem

Messages passed between services as JSON with no type safety lead to runtime deserialization failures and contract mismatches.

## Example

### ✅ Type-Safe Messages

```csharp
// Message contracts as types
[DataContract]
public sealed record OrderCreatedMessage
{
    [DataMember(Name = "order_id", IsRequired = true)]
    public string OrderId { get; init; }
    
    [DataMember(Name = "customer_id", IsRequired = true)]
    public string CustomerId { get; init; }
    
    [DataMember(Name = "total", IsRequired = true)]
    public decimal Total { get; init; }
    
    [DataMember(Name = "created_at", IsRequired = true)]
    public string CreatedAt { get; init; }
    
    public static OrderCreatedMessage FromDomain(Order order) =>
        new OrderCreatedMessage
        {
            OrderId = order.Id.Value.ToString(),
            CustomerId = order.CustomerId.Value.ToString(),
            Total = order.Total.Amount,
            CreatedAt = DateTimeOffset.UtcNow.ToString("O")
        };
    
    public Result<(OrderId, CustomerId, Money), string> ToDomain()
    {
        if (!Guid.TryParse(OrderId, out var orderId))
            return Result<(OrderId, CustomerId, Money), string>.Failure("Invalid OrderId");
        
        if (!Guid.TryParse(CustomerId, out var customerId))
            return Result<(OrderId, CustomerId, Money), string>.Failure("Invalid CustomerId");
        
        return Result<(OrderId, CustomerId, Money), string>.Success((
            new OrderId(orderId),
            new CustomerId(customerId),
            Money.USD(Total)));
    }
}
```

## See Also

- [DTO vs Domain Boundary](./dto-domain-boundary.md)
- [Anti-Corruption Layer](./anti-corruption-layer.md)
