# Enforcing Call Order with Types

> Runtime guards (boolean flags + exceptions) to enforce method call sequences—use types instead.

## Problem

When business workflows require operations in a specific order, runtime checks (boolean flags + exceptions) are error-prone. Invalid sequences compile successfully, guard clauses proliferate, and contradictory states become representable.

## Example

### ❌ Before

```csharp
public sealed class OrderFulfillmentService
{
    bool inventoryReserved;
    bool paymentCaptured;
    bool shipped;

    public void ReserveInventory(OrderId orderId, IReadOnlyList<Sku> itemSkus)
    {
        Console.WriteLine($"Reserving inventory for order {orderId}: {string.Join(",", itemSkus)}");
        inventoryReserved = true;
    }

    public void CapturePayment(OrderId orderId, OrderValue value)
    {
        if (!inventoryReserved)
            throw new InvalidOperationException("Cannot capture payment before reserving inventory.");

        Console.WriteLine($"Capturing {value} for order {orderId}");
        paymentCaptured = true;
    }

    public void ShipProduct(OrderId orderId, Carrier carrier)
    {
        if (!inventoryReserved)
            throw new InvalidOperationException("Cannot ship: inventory not reserved.");
        if (!paymentCaptured)
            throw new InvalidOperationException("Cannot ship: payment not captured.");

        Console.WriteLine($"Shipping order {orderId} via {carrier}");
        shipped = true;
    }
}

// All compile; some fail only at runtime
var svc = new OrderFulfillmentService();
svc.ShipProduct("O-123", "UPS");                  // Runtime exception
svc.CapturePayment("O-123", 99.00m);              // Runtime exception
svc.ReserveInventory("O-123", new[] { "SKU-1" }); // OK
svc.ReserveInventory("O-123", new[] { "SKU-1" }); // Double reservation—no protection!
```

### ✅ After

```csharp
// Each state is a distinct type; only valid transitions are possible

public sealed record PendingOrder(OrderId Id, IReadOnlyList<Sku> Items)
{
    public ReservedOrder ReserveInventory()
    {
        Console.WriteLine($"Reserving inventory for order {Id}: {string.Join(",", Items)}");
        return new ReservedOrder(Id, Items);
    }
}

public sealed record ReservedOrder(OrderId Id, IReadOnlyList<Sku> Items)
{
    public PaidOrder CapturePayment(OrderValue value)
    {
        Console.WriteLine($"Capturing {value} for order {Id}");
        return new PaidOrder(Id, Items, value);
    }
}

public sealed record PaidOrder(OrderId Id, IReadOnlyList<Sku> Items, OrderValue Value)
{
    public ShippedOrder Ship(Carrier carrier)
    {
        Console.WriteLine($"Shipping order {Id} via {carrier}");
        return new ShippedOrder(Id, Items, Value, carrier);
    }
}

public sealed record ShippedOrder(OrderId Id, IReadOnlyList<Sku> Items, OrderValue Value, Carrier Carrier);

// Usage: only valid sequences compile
var pending = new PendingOrder("O-123", new[] { "SKU-1" });
var reserved = pending.ReserveInventory();
var paid = reserved.CapturePayment(99.00m);
var shipped = paid.Ship("UPS");

// These don't compile:
// pending.CapturePayment(99.00m);  // Error: PendingOrder has no CapturePayment method
// pending.Ship("UPS");             // Error: PendingOrder has no Ship method
// reserved.Ship("UPS");            // Error: ReservedOrder has no Ship method
```

## Why It's a Problem

1. **Hidden protocol**: Call order exists only in comments and guard clauses—easy to miss or misunderstand.

2. **Runtime guard fatigue**: Every method accumulates defensive `if` checks; callers still attempt invalid sequences.

3. **Invalid states are representable**: `paymentCaptured == true` while `inventoryReserved == false` can exist in memory.

4. **Harder testing**: Need separate tests for each invalid ordering; compiler provides no help.

## Symptoms

- Boolean flags controlling whether a method may proceed
- Comments describing required prior calls ("must call X first")
- Methods throwing `InvalidOperationException` when preconditions aren't met
- Ability to represent contradictory state combinations
- Multiple methods that must be called in sequence with no compile-time enforcement

## Benefits

- **Compile-time enforcement**: Invalid sequences don't compile—no runtime order guards needed
- **Impossible invalid states**: Each type only exposes valid next steps
- **Self-documenting API**: Method availability communicates allowed transitions
- **Fewer tests**: No need to test invalid orderings; the type system prevents them
- **Clearer domain model**: State machine is explicit in the type hierarchy

## Variation: Builder Pattern

For simpler cases, a fluent builder can enforce ordering:

```csharp
public sealed class OrderBuilder
{
    private OrderBuilder() { }
    
    public static IRequiresInventory Create(OrderId id) => new Builder(id);
    
    public interface IRequiresInventory
    {
        IRequiresPayment ReserveInventory(IReadOnlyList<Sku> items);
    }
    
    public interface IRequiresPayment
    {
        IReadyToShip CapturePayment(OrderValue value);
    }
    
    public interface IReadyToShip
    {
        ShippedOrder Ship(Carrier carrier);
    }
    
    private class Builder : IRequiresInventory, IRequiresPayment, IReadyToShip
    {
        private readonly OrderId _id;
        private IReadOnlyList<Sku>? _items;
        private OrderValue? _value;
        
        public Builder(OrderId id) => _id = id;
        
        public IRequiresPayment ReserveInventory(IReadOnlyList<Sku> items)
        {
            _items = items;
            return this;
        }
        
        public IReadyToShip CapturePayment(OrderValue value)
        {
            _value = value;
            return this;
        }
        
        public ShippedOrder Ship(Carrier carrier) => 
            new(_id, _items!, _value!.Value, carrier);
    }
}

// Usage
var order = OrderBuilder
    .Create("O-123")
    .ReserveInventory(new[] { "SKU-1" })
    .CapturePayment(99.00m)
    .Ship("UPS");
```

## See Also

- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md)
- [Static Factory Methods](./static-factory-methods.md)
