# Type-Safe Event Sourcing (Events as Source of Truth)

> Storing only current state loses history—use event sourcing with typed events to maintain complete audit trail and enable time travel.

## Problem

Traditional CRUD stores only current state, making it impossible to reconstruct how entities reached their current state or to replay events.

## Example

### ✅ Type-Safe Event Store

```csharp
// Events as immutable types
public abstract record OrderEvent
{
    public OrderId OrderId { get; init; }
    public DateTimeOffset Timestamp { get; init; }
    
    public sealed record OrderCreated(
        OrderId OrderId,
        CustomerId CustomerId,
        DateTimeOffset Timestamp) : OrderEvent;
    
    public sealed record ItemAdded(
        OrderId OrderId,
        ProductId ProductId,
        int Quantity,
        Money UnitPrice,
        DateTimeOffset Timestamp) : OrderEvent;
    
    public sealed record OrderSubmitted(
        OrderId OrderId,
        DateTimeOffset Timestamp) : OrderEvent;
    
    public sealed record OrderPaid(
        OrderId OrderId,
        PaymentId PaymentId,
        DateTimeOffset Timestamp) : OrderEvent;
}

// Aggregate rebuilt from events
public sealed class Order
{
    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderLine> _lines = new();
    
    public static Order FromEvents(IEnumerable<OrderEvent> events)
    {
        var order = new Order();
        foreach (var evt in events)
        {
            order.Apply(evt);
        }
        return order;
    }
    
    private void Apply(OrderEvent evt)
    {
        switch (evt)
        {
            case OrderEvent.OrderCreated e:
                Id = e.OrderId;
                CustomerId = e.CustomerId;
                Status = OrderStatus.Draft;
                break;
            
            case OrderEvent.ItemAdded e:
                _lines.Add(new OrderLine(e.ProductId, e.Quantity, e.UnitPrice));
                break;
            
            case OrderEvent.OrderSubmitted e:
                Status = OrderStatus.Submitted;
                break;
            
            case OrderEvent.OrderPaid e:
                Status = OrderStatus.Paid;
                break;
        }
    }
}
```

## See Also

- [Domain Events](./domain-events.md)
- [Type-Safe State Transitions](./type-safe-state-transitions.md)
