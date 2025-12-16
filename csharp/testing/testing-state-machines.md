# Testing State Machines

> Verify state transitions—test valid transitions succeed and invalid ones fail.

## Problem

State machines have complex transition rules that must be enforced. Without comprehensive tests, invalid transitions can occur, leading to corrupted state.

## Pattern

Test all valid transitions, verify invalid transitions are prevented, and ensure state-dependent behavior works correctly.

## Example

### ❌ Before - Incomplete State Testing

```csharp
[Fact]
public void ProcessOrder_Works()
{
    var order = OrderMother.PendingOrder();
    order.Process();
    Assert.Equal(OrderStatus.Processing, order.Status);
}
// ❌ What about other transitions? Invalid transitions?
```

### ✅ After - Comprehensive State Machine Testing

```csharp
[Theory]
[InlineData(OrderStatus.Pending, OrderStatus.Confirmed)]
[InlineData(OrderStatus.Confirmed, OrderStatus.Processing)]
[InlineData(OrderStatus.Processing, OrderStatus.Shipped)]
public void TransitionTo_ValidTransition_Succeeds(OrderStatus from, OrderStatus to)
{
    // Arrange
    var order = OrderMother.OrderInStatus(from);

    // Act
    var result = order.TransitionTo(to);

    // Assert
    Assert.True(result.IsSuccess);
    Assert.Equal(to, order.Status);
}

[Theory]
[InlineData(OrderStatus.Shipped, OrderStatus.Pending)]
[InlineData(OrderStatus.Delivered, OrderStatus.Processing)]
[InlineData(OrderStatus.Cancelled, OrderStatus.Shipped)]
public void TransitionTo_InvalidTransition_Fails(OrderStatus from, OrderStatus to)
{
    // Arrange
    var order = OrderMother.OrderInStatus(from);
    var originalStatus = order.Status;

    // Act
    var result = order.TransitionTo(to);

    // Assert
    Assert.True(result.IsFailure);
    Assert.Equal(originalStatus, order.Status); // Status unchanged
}
```

## Testing Valid Transitions

### Explicit Transition Methods

```csharp
public class OrderStateTests
{
    [Fact]
    public void Confirm_FromPending_TransitionsToConfirmed()
    {
        // Arrange
        var order = OrderMother.PendingOrder();

        // Act
        var result = order.Confirm();

        // Assert
        Assert.True(result.IsSuccess);
        Assert.Equal(OrderStatus.Confirmed, order.Status);
    }

    [Fact]
    public void Process_FromConfirmed_TransitionsToProcessing()
    {
        // Arrange
        var order = OrderMother.ConfirmedOrder();

        // Act
        var result = order.Process();

        // Assert
        Assert.True(result.IsSuccess);
        Assert.Equal(OrderStatus.Processing, order.Status);
    }

    [Fact]
    public void Ship_FromProcessing_TransitionsToShipped()
    {
        // Arrange
        var order = OrderMother.ProcessingOrder();
        var trackingNumber = TrackingNumber.Create("TRACK123");

        // Act
        var result = order.Ship(trackingNumber);

        // Assert
        Assert.True(result.IsSuccess);
        Assert.Equal(OrderStatus.Shipped, order.Status);
        Assert.Equal(trackingNumber, order.TrackingNumber);
    }
}
```

### Transition Matrix Testing

```csharp
public class OrderTransitionMatrixTests
{
    public static IEnumerable<object[]> ValidTransitions =>
        new List<object[]>
        {
            new object[] { OrderStatus.Pending, OrderStatus.Confirmed },
            new object[] { OrderStatus.Pending, OrderStatus.Cancelled },
            new object[] { OrderStatus.Confirmed, OrderStatus.Processing },
            new object[] { OrderStatus.Confirmed, OrderStatus.Cancelled },
            new object[] { OrderStatus.Processing, OrderStatus.Shipped },
            new object[] { OrderStatus.Shipped, OrderStatus.Delivered },
        };

    public static IEnumerable<object[]> InvalidTransitions =>
        new List<object[]>
        {
            new object[] { OrderStatus.Delivered, OrderStatus.Pending },
            new object[] { OrderStatus.Delivered, OrderStatus.Processing },
            new object[] { OrderStatus.Cancelled, OrderStatus.Processing },
            new object[] { OrderStatus.Shipped, OrderStatus.Confirmed },
            new object[] { OrderStatus.Processing, OrderStatus.Delivered }, // Skip shipped
        };

    [Theory]
    [MemberData(nameof(ValidTransitions))]
    public void TransitionTo_ValidTransition_Succeeds(OrderStatus from, OrderStatus to)
    {
        var order = OrderMother.OrderInStatus(from);
        
        var result = order.TransitionTo(to);
        
        Assert.True(result.IsSuccess);
        Assert.Equal(to, order.Status);
    }

    [Theory]
    [MemberData(nameof(InvalidTransitions))]
    public void TransitionTo_InvalidTransition_ReturnsFailure(OrderStatus from, OrderStatus to)
    {
        var order = OrderMother.OrderInStatus(from);
        var originalStatus = order.Status;
        
        var result = order.TransitionTo(to);
        
        Assert.True(result.IsFailure);
        Assert.Equal(originalStatus, order.Status);
        Assert.Contains("cannot transition", result.Error, StringComparison.OrdinalIgnoreCase);
    }
}
```

## Testing State-Dependent Behavior

```csharp
public class StateDependent BehaviorTests
{
    [Fact]
    public void AddItem_ToPendingOrder_Succeeds()
    {
        // Can only modify pending orders
        var order = OrderMother.PendingOrder();
        var item = OrderItem.Create(ProductId.New(), quantity: 1, Money.USD(10m));

        var result = order.AddItem(item);

        Assert.True(result.IsSuccess);
    }

    [Theory]
    [InlineData(OrderStatus.Confirmed)]
    [InlineData(OrderStatus.Processing)]
    [InlineData(OrderStatus.Shipped)]
    public void AddItem_ToNonPendingOrder_Fails(OrderStatus status)
    {
        // Cannot modify orders in other states
        var order = OrderMother.OrderInStatus(status);
        var item = OrderItem.Create(ProductId.New(), quantity: 1, Money.USD(10m));

        var result = order.AddItem(item);

        Assert.True(result.IsFailure);
        Assert.Contains("cannot add items", result.Error, StringComparison.OrdinalIgnoreCase);
    }

    [Fact]
    public void Cancel_FromPendingOrConfirmed_Succeeds()
    {
        var pendingOrder = OrderMother.PendingOrder();
        var confirmedOrder = OrderMother.ConfirmedOrder();

        Assert.True(pendingOrder.Cancel().IsSuccess);
        Assert.True(confirmedOrder.Cancel().IsSuccess);
    }

    [Theory]
    [InlineData(OrderStatus.Shipped)]
    [InlineData(OrderStatus.Delivered)]
    public void Cancel_FromShippedOrDelivered_Fails(OrderStatus status)
    {
        var order = OrderMother.OrderInStatus(status);

        var result = order.Cancel();

        Assert.True(result.IsFailure);
    }
}
```

## Testing State Machine Invariants

```csharp
public class StateInvariantsTests
{
    [Fact]
    public void ShippedOrder_MustHaveTrackingNumber()
    {
        // Invariant: Shipped orders must have tracking number
        var order = OrderMother.ProcessingOrder();

        var resultWithoutTracking = order.Ship(null!);
        Assert.True(resultWithoutTracking.IsFailure);

        var resultWithTracking = order.Ship(TrackingNumber.Create("TRACK123"));
        Assert.True(resultWithTracking.IsSuccess);
        Assert.NotNull(order.TrackingNumber);
    }

    [Fact]
    public void DeliveredOrder_MustHaveShipmentDate()
    {
        // Invariant: Can only deliver shipped orders
        var order = OrderMother.ShippedOrder();

        var result = order.Deliver();

        Assert.True(result.IsSuccess);
        Assert.NotNull(order.ShippedAt);
        Assert.NotNull(order.DeliveredAt);
        Assert.True(order.DeliveredAt >= order.ShippedAt);
    }
}
```

## Testing Type-Safe State Machines

Using discriminated unions for type-safe states:

```csharp
public abstract record OrderState
{
    public sealed record Pending : OrderState;
    public sealed record Confirmed(DateTime ConfirmedAt) : OrderState;
    public sealed record Processing(DateTime ProcessingStartedAt) : OrderState;
    public sealed record Shipped(TrackingNumber Tracking, DateTime ShippedAt) : OrderState;
    public sealed record Delivered(DateTime DeliveredAt) : OrderState;
    public sealed record Cancelled(string Reason, DateTime CancelledAt) : OrderState;
}

public class TypeSafeStateTests
{
    [Fact]
    public void Confirm_FromPending_ReturnsConfirmedState()
    {
        // Arrange
        var order = new Order(OrderId.New(), new OrderState.Pending());

        // Act
        var newState = order.Confirm();

        // Assert
        Assert.IsType<OrderState.Confirmed>(newState);
        Assert.NotNull(((OrderState.Confirmed)newState).ConfirmedAt);
    }

    [Fact]
    public void Ship_FromProcessing_ReturnsShippedStateWithTracking()
    {
        // Arrange
        var processingState = new OrderState.Processing(DateTime.UtcNow);
        var order = new Order(OrderId.New(), processingState);
        var tracking = TrackingNumber.Create("TRACK123");

        // Act
        var newState = order.Ship(tracking);

        // Assert
        var shippedState = Assert.IsType<OrderState.Shipped>(newState);
        Assert.Equal(tracking, shippedState.Tracking);
        Assert.True(shippedState.ShippedAt <= DateTime.UtcNow);
    }

    [Fact]
    public void PatternMatch_HandlesAllStates()
    {
        // Arrange
        var states = new OrderState[]
        {
            new OrderState.Pending(),
            new OrderState.Confirmed(DateTime.UtcNow),
            new OrderState.Processing(DateTime.UtcNow),
            new OrderState.Shipped(TrackingNumber.Create("123"), DateTime.UtcNow),
            new OrderState.Delivered(DateTime.UtcNow),
            new OrderState.Cancelled("Customer request", DateTime.UtcNow),
        };

        // Act & Assert - Verify all states are handled
        foreach (var state in states)
        {
            var description = state switch
            {
                OrderState.Pending => "Pending",
                OrderState.Confirmed => "Confirmed",
                OrderState.Processing => "Processing",
                OrderState.Shipped => "Shipped",
                OrderState.Delivered => "Delivered",
                OrderState.Cancelled => "Cancelled",
                _ => throw new InvalidOperationException("Unhandled state")
            };
            
            Assert.NotNull(description);
        }
    }
}
```

## Testing State History

```csharp
public class StateHistoryTests
{
    [Fact]
    public void StateChanges_AreRecorded()
    {
        // Arrange
        var order = OrderMother.PendingOrder();

        // Act
        order.Confirm();
        order.Process();
        order.Ship(TrackingNumber.Create("TRACK123"));

        // Assert
        var history = order.StateHistory;
        Assert.Equal(4, history.Count); // Pending, Confirmed, Processing, Shipped
        
        Assert.Equal(OrderStatus.Pending, history[0].Status);
        Assert.Equal(OrderStatus.Confirmed, history[1].Status);
        Assert.Equal(OrderStatus.Processing, history[2].Status);
        Assert.Equal(OrderStatus.Shipped, history[3].Status);
    }

    [Fact]
    public void StateTransitions_RecordTimestamps()
    {
        var order = OrderMother.PendingOrder();
        var beforeConfirm = DateTime.UtcNow;

        order.Confirm();

        var history = order.StateHistory;
        var confirmedEntry = history.First(h => h.Status == OrderStatus.Confirmed);
        
        Assert.True(confirmedEntry.Timestamp >= beforeConfirm);
        Assert.True(confirmedEntry.Timestamp <= DateTime.UtcNow);
    }
}
```

## Testing Concurrent State Changes

```csharp
public class ConcurrentStateTests
{
    [Fact]
    public void ConcurrentTransitions_AreSerializedCorrectly()
    {
        // Arrange
        var order = OrderMother.PendingOrder();
        var tasks = new List<Task<Result<Unit>>>();

        // Act - Try to confirm multiple times concurrently
        for (int i = 0; i < 10; i++)
        {
            tasks.Add(Task.Run(() => order.Confirm()));
        }
        Task.WaitAll(tasks.ToArray());

        // Assert - Only one should succeed
        var successCount = tasks.Count(t => t.Result.IsSuccess);
        Assert.Equal(1, successCount);
        Assert.Equal(OrderStatus.Confirmed, order.Status);
    }
}
```

## Testing Phantom States

Using phantom types to make invalid states unrepresentable:

```csharp
// Using type system to enforce valid transitions
public sealed class Order<TState> where TState : IOrderState
{
    public Order<Confirmed> Confirm() where TState : Pending
    {
        // Compile-time guarantee: can only confirm pending orders
        return new Order<Confirmed>(/* ... */);
    }

    public Order<Shipped> Ship(TrackingNumber tracking) where TState : Processing
    {
        // Compile-time guarantee: can only ship processing orders
        return new Order<Shipped>(/* ... */);
    }
}

public class PhantomStateTests
{
    [Fact]
    public void TypeSafeTransitions_EnforceValidSequence()
    {
        // Arrange
        var pending = new Order<Pending>(OrderId.New());

        // Act
        var confirmed = pending.Confirm();
        var processing = confirmed.Process();
        var shipped = processing.Ship(TrackingNumber.Create("123"));

        // These would not compile:
        // pending.Ship(tracking);  // ❌ Can't ship pending order
        // confirmed.Deliver();      // ❌ Can't deliver confirmed order
        
        Assert.NotNull(shipped);
    }
}
```

## Guidelines

1. **Test all valid transitions** explicitly
2. **Test all invalid transitions** are prevented
3. **Verify state-dependent behavior** works correctly
4. **Test state invariants** are maintained
5. **Use parameterized tests** for transition matrices
6. **Consider type-safe state machines** for compile-time safety

## Benefits

1. **Correctness**: Ensures state transitions are valid
2. **Safety**: Invalid transitions caught early
3. **Documentation**: Tests show allowed transitions
4. **Confidence**: Know state machine works correctly
5. **Maintainability**: Easy to add new states

## See Also

- [Testing Domain Invariants](./testing-domain-invariants.md) — state-dependent rules
- [Testing Discriminated Unions](./testing-discriminated-unions.md) — type-safe states
- [Type-Level State Machines](../patterns/type-level-state-machines.md) — design pattern
- [Enum State Machine](../patterns/enum-state-machine.md) — implementation pattern
