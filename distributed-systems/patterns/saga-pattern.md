# Saga Pattern

> Long-running transactions broken into local transactions with compensating actions for rollback—achieves eventual consistency without distributed locks.

## Problem

Traditional ACID transactions don't work well in distributed systems. A single business transaction might span multiple services, each with its own database. Distributed transactions (2PC) are slow, reduce availability, and create tight coupling. The Saga pattern provides an alternative: break the transaction into steps, and if something fails, undo what you did.

## The Saga Approach

```
Traditional Transaction                 Saga Pattern
      
┌──────────────────┐                   Step 1: Reserve inventory
│ BEGIN TRANSACTION│                   ┌─────────────────────┐
├──────────────────┤                   │ Reserve succeeded   │
│ Reserve inventory│                   └──────────┬──────────┘
│ Charge payment   │                              │
│ Create shipment  │                   Step 2: Charge payment
│ Send email       │                   ┌─────────┴──────────┐
├──────────────────┤                   │ Payment failed!    │
│ COMMIT           │                   └──────────┬─────────┘
└──────────────────┘                              │
                                        Step 3: Compensate
If ANY step fails,                      ┌─────────┴──────────┐
ROLLBACK everything                     │ Unreserve inventory│
(requires locks)                        │ (compensation)     │
                                        └────────────────────┘
```

## Example

### ❌ Distributed Transaction (2PC)

```csharp
public class OrderService
{
    public async Task<Result<Order, OrderError>> CreateOrder(CreateOrderRequest request)
    {
        // Requires distributed transaction coordinator
        using var transaction = await _transactionCoordinator.BeginAsync();
        
        try
        {
            // Step 1: Reserve inventory (locks resources)
            await _inventoryService.ReserveAsync(
                transaction,
                request.Items);
            
            // Step 2: Charge payment (locks resources)
            await _paymentService.ChargeAsync(
                transaction,
                request.CustomerId,
                request.Total);
            
            // Step 3: Create shipment (locks resources)
            await _shipmentService.CreateAsync(
                transaction,
                request.Address);
            
            // All services must vote to commit
            await transaction.CommitAsync();  // Blocking, slow
            
            return Result.Success(new Order(request));
        }
        catch (Exception ex)
        {
            // Rollback across all services
            await transaction.RollbackAsync();  // Blocking, slow
            return Result.Failure<Order, OrderError>(
                new OrderError.TransactionFailed(ex.Message));
        }
    }
}
```

**Problems:**
- Requires all services to support distributed transactions
- Locks resources across multiple services (poor throughput)
- Reduces availability (all services must be up)
- Tight coupling between services

### ✅ Saga Pattern: Choreography-Based

```csharp
/// <summary>
/// Events that drive the saga.
/// </summary>
public abstract record OrderSagaEvent
{
    public sealed record OrderCreated(
        OrderId OrderId,
        CustomerId CustomerId,
        List<OrderItem> Items,
        Money Total,
        Address ShippingAddress) : OrderSagaEvent;
    
    public sealed record InventoryReserved(
        OrderId OrderId,
        ReservationId ReservationId) : OrderSagaEvent;
    
    public sealed record InventoryReservationFailed(
        OrderId OrderId,
        string Reason) : OrderSagaEvent;
    
    public sealed record PaymentCharged(
        OrderId OrderId,
        PaymentId PaymentId) : OrderSagaEvent;
    
    public sealed record PaymentFailed(
        OrderId OrderId,
        string Reason) : OrderSagaEvent;
    
    public sealed record ShipmentCreated(
        OrderId OrderId,
        ShipmentId ShipmentId) : OrderSagaEvent;
    
    public sealed record OrderCompleted(OrderId OrderId) : OrderSagaEvent;
    
    public sealed record OrderCancelled(
        OrderId OrderId,
        string Reason) : OrderSagaEvent;
}

/// <summary>
/// Order service initiates the saga.
/// </summary>
public class OrderService
{
    private readonly IEventBus _eventBus;
    private readonly IOrderRepository _orders;
    
    public async Task<Result<OrderId, OrderError>> CreateOrder(CreateOrderRequest request)
    {
        // Create order in Pending state
        var orderId = OrderId.New();
        var order = new Order
        {
            Id = orderId,
            Status = OrderStatus.Pending,
            CustomerId = request.CustomerId,
            Items = request.Items,
            Total = request.Total
        };
        
        await _orders.SaveAsync(order);
        
        // Publish event to start saga
        await _eventBus.PublishAsync(new OrderSagaEvent.OrderCreated(
            orderId,
            request.CustomerId,
            request.Items,
            request.Total,
            request.ShippingAddress));
        
        return Result.Success(orderId);
    }
}

/// <summary>
/// Inventory service reserves items and publishes result.
/// </summary>
public class InventoryService
{
    private readonly IEventBus _eventBus;
    private readonly IInventoryRepository _inventory;
    
    public async Task HandleAsync(OrderSagaEvent.OrderCreated @event)
    {
        try
        {
            // Try to reserve inventory
            var reservationId = await _inventory.ReserveAsync(@event.Items);
            
            // Success—publish event
            await _eventBus.PublishAsync(new OrderSagaEvent.InventoryReserved(
                @event.OrderId,
                reservationId));
        }
        catch (InsufficientInventoryException ex)
        {
            // Failure—publish failure event
            await _eventBus.PublishAsync(new OrderSagaEvent.InventoryReservationFailed(
                @event.OrderId,
                ex.Message));
        }
    }
    
    /// <summary>
    /// Compensating action: unreserve inventory.
    /// </summary>
    public async Task HandleAsync(OrderSagaEvent.PaymentFailed @event)
    {
        // Payment failed, compensate by unreserving inventory
        await _inventory.UnreserveAsync(@event.OrderId);
    }
}

/// <summary>
/// Payment service charges customer and publishes result.
/// </summary>
public class PaymentService
{
    private readonly IEventBus _eventBus;
    private readonly IPaymentGateway _gateway;
    
    public async Task HandleAsync(OrderSagaEvent.InventoryReserved @event)
    {
        try
        {
            // Charge customer
            var paymentId = await _gateway.ChargeAsync(
                @event.OrderId,
                @event.CustomerId,
                @event.Total);
            
            // Success—publish event
            await _eventBus.PublishAsync(new OrderSagaEvent.PaymentCharged(
                @event.OrderId,
                paymentId));
        }
        catch (PaymentFailedException ex)
        {
            // Failure—publish failure event
            await _eventBus.PublishAsync(new OrderSagaEvent.PaymentFailed(
                @event.OrderId,
                ex.Message));
        }
    }
}

/// <summary>
/// Shipment service creates shipment and completes saga.
/// </summary>
public class ShipmentService
{
    private readonly IEventBus _eventBus;
    private readonly IShipmentRepository _shipments;
    
    public async Task HandleAsync(OrderSagaEvent.PaymentCharged @event)
    {
        // Create shipment
        var shipmentId = await _shipments.CreateAsync(@event.OrderId);
        
        await _eventBus.PublishAsync(new OrderSagaEvent.ShipmentCreated(
            @event.OrderId,
            shipmentId));
        
        // Saga completed successfully
        await _eventBus.PublishAsync(new OrderSagaEvent.OrderCompleted(
            @event.OrderId));
    }
}

/// <summary>
/// Order service updates status based on saga outcome.
/// </summary>
public class OrderStatusHandler
{
    private readonly IOrderRepository _orders;
    
    public async Task HandleAsync(OrderSagaEvent.OrderCompleted @event)
    {
        var order = await _orders.GetAsync(@event.OrderId);
        order.Status = OrderStatus.Confirmed;
        await _orders.SaveAsync(order);
    }
    
    public async Task HandleAsync(OrderSagaEvent.InventoryReservationFailed @event)
    {
        var order = await _orders.GetAsync(@event.OrderId);
        order.Status = OrderStatus.Cancelled;
        order.CancellationReason = @event.Reason;
        await _orders.SaveAsync(order);
    }
    
    public async Task HandleAsync(OrderSagaEvent.PaymentFailed @event)
    {
        var order = await _orders.GetAsync(@event.OrderId);
        order.Status = OrderStatus.Cancelled;
        order.CancellationReason = @event.Reason;
        await _orders.SaveAsync(order);
    }
}
```

### ✅ Saga Pattern: Orchestration-Based

```csharp
/// <summary>
/// Commands sent by orchestrator to services.
/// </summary>
public abstract record OrderSagaCommand
{
    public sealed record ReserveInventory(
        OrderId OrderId,
        List<OrderItem> Items) : OrderSagaCommand;
    
    public sealed record ChargePayment(
        OrderId OrderId,
        CustomerId CustomerId,
        Money Total) : OrderSagaCommand;
    
    public sealed record CreateShipment(
        OrderId OrderId,
        Address ShippingAddress) : OrderSagaCommand;
    
    public sealed record UnreserveInventory(
        OrderId OrderId) : OrderSagaCommand;
    
    public sealed record RefundPayment(
        OrderId OrderId,
        PaymentId PaymentId) : OrderSagaCommand;
}

/// <summary>
/// Orchestrator coordinates the saga.
/// </summary>
public class OrderSagaOrchestrator
{
    private readonly IEventBus _eventBus;
    private readonly ISagaRepository _sagas;
    
    public async Task HandleAsync(OrderSagaEvent.OrderCreated @event)
    {
        // Create saga state
        var saga = new OrderSaga
        {
            OrderId = @event.OrderId,
            State = SagaState.ReservingInventory,
            CustomerId = @event.CustomerId,
            Items = @event.Items,
            Total = @event.Total,
            ShippingAddress = @event.ShippingAddress
        };
        
        await _sagas.SaveAsync(saga);
        
        // Send first command
        await _eventBus.PublishAsync(new OrderSagaCommand.ReserveInventory(
            @event.OrderId,
            @event.Items));
    }
    
    public async Task HandleAsync(OrderSagaEvent.InventoryReserved @event)
    {
        var saga = await _sagas.GetAsync(@event.OrderId);
        saga.State = SagaState.ChargingPayment;
        saga.ReservationId = @event.ReservationId;
        await _sagas.SaveAsync(saga);
        
        // Send next command
        await _eventBus.PublishAsync(new OrderSagaCommand.ChargePayment(
            saga.OrderId,
            saga.CustomerId,
            saga.Total));
    }
    
    public async Task HandleAsync(OrderSagaEvent.PaymentCharged @event)
    {
        var saga = await _sagas.GetAsync(@event.OrderId);
        saga.State = SagaState.CreatingShipment;
        saga.PaymentId = @event.PaymentId;
        await _sagas.SaveAsync(saga);
        
        // Send next command
        await _eventBus.PublishAsync(new OrderSagaCommand.CreateShipment(
            saga.OrderId,
            saga.ShippingAddress));
    }
    
    public async Task HandleAsync(OrderSagaEvent.ShipmentCreated @event)
    {
        var saga = await _sagas.GetAsync(@event.OrderId);
        saga.State = SagaState.Completed;
        await _sagas.SaveAsync(saga);
        
        // Saga completed successfully
        await _eventBus.PublishAsync(new OrderSagaEvent.OrderCompleted(
            @event.OrderId));
    }
    
    /// <summary>
    /// Handle failure: trigger compensating actions.
    /// </summary>
    public async Task HandleAsync(OrderSagaEvent.PaymentFailed @event)
    {
        var saga = await _sagas.GetAsync(@event.OrderId);
        saga.State = SagaState.Compensating;
        await _sagas.SaveAsync(saga);
        
        // Compensate: unreserve inventory
        await _eventBus.PublishAsync(new OrderSagaCommand.UnreserveInventory(
            saga.OrderId));
        
        saga.State = SagaState.Cancelled;
        saga.CancellationReason = @event.Reason;
        await _sagas.SaveAsync(saga);
    }
}

public enum SagaState
{
    ReservingInventory,
    ChargingPayment,
    CreatingShipment,
    Completed,
    Compensating,
    Cancelled
}
```

## Choreography vs. Orchestration

### Choreography

**Pros:**
- No central coordinator (better availability)
- Services are loosely coupled
- Easy to add new participants

**Cons:**
- Hard to understand flow (distributed logic)
- Difficult to track saga state
- Cyclic dependencies possible

### Orchestration

**Pros:**
- Central control (easy to understand)
- Clear saga state tracking
- Easier to handle timeouts and retries

**Cons:**
- Central orchestrator is single point of failure
- Orchestrator can become complex
- Services coupled to orchestrator

## Handling Compensations

Not all actions can be perfectly undone:

```csharp
/// <summary>
/// Compensating action for sending email (can't unsend).
/// </summary>
public class EmailCompensation : IEventHandler<OrderSagaEvent.OrderCancelled>
{
    private readonly IEmailService _emailService;
    
    public async Task HandleAsync(OrderSagaEvent.OrderCancelled @event)
    {
        // Can't unsend email, but can send cancellation notice
        await _emailService.SendAsync(new Email
        {
            To = @event.CustomerId,
            Subject = "Order Cancelled",
            Body = $"Your order {@event.OrderId} was cancelled: {@event.Reason}"
        });
    }
}
```

## Why It's a Problem (Not Using Sagas)

1. **Distributed transactions don't scale**: 2PC is slow and reduces availability
2. **All-or-nothing doesn't work**: In distributed systems, partial failures are common
3. **Locks are expensive**: Holding locks across services kills throughput

## Symptoms

- Distributed transactions causing timeouts
- Services becoming unavailable together
- Inconsistent state after failures
- Manual cleanup required after errors

## Benefits

- **No distributed locks**: Each service operates independently
- **Better availability**: Services don't wait for each other
- **Graceful failure handling**: Compensations handle partial failures
- **Loose coupling**: Services communicate via events

## See Also

- [Event-Driven Architecture](./event-driven-architecture.md) — event-based communication
- [Eventual Consistency](./eventual-consistency.md) — accepting temporary inconsistency
- [Two-Phase Commit](./two-phase-commit.md) — alternative distributed transaction approach
- [CQRS](./cqrs.md) — often combined with sagas
