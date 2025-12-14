# Saga Pattern (Long-Running Distributed Transactions)

> Distributed operations fail midway leaving inconsistent state—use sagas to coordinate multi-step workflows with compensation logic encoded in types.

## Problem

In distributed systems, operations span multiple services and can fail at any step. Traditional ACID transactions don't work across service boundaries. Without proper coordination, systems end up in inconsistent states.

## Example

### ❌ Before

```csharp
// Distributed operation without saga coordination
public class OrderService
{
    private readonly IInventoryService _inventory;
    private readonly IPaymentService _payment;
    private readonly IShippingService _shipping;
    
    public async Task<bool> PlaceOrder(Order order)
    {
        try
        {
            // Step 1: Reserve inventory
            await _inventory.ReserveAsync(order.Items);
            
            // Step 2: Process payment
            await _payment.ChargeAsync(order.CustomerId, order.Total);
            
            // Step 3: Create shipment
            await _shipping.CreateShipmentAsync(order);
            
            return true;
        }
        catch (Exception ex)
        {
            // BUG: What if payment succeeded but shipping failed?
            // Inventory is still reserved and payment is charged!
            return false;
        }
    }
}
```

**Problems:**
- No compensation if later steps fail
- Inconsistent state when failures occur
- Manual rollback logic error-prone
- Hard to track saga progress
- Can't resume after crashes

### ✅ After (Saga with Type-Safe Steps)

```csharp
// ============================================
// SAGA STATE (Type-safe state machine)
// ============================================
namespace Orders.Sagas
{
    /// <summary>
    /// Each saga state is a distinct type.
    /// State encodes what has been done and what can be done next.
    /// </summary>
    public abstract record PlaceOrderSagaState
    {
        private PlaceOrderSagaState() { }
        
        // Initial state
        public sealed record Started(
            SagaId SagaId,
            OrderId OrderId,
            CustomerId CustomerId,
            IReadOnlyList<OrderLine> Items,
            Money Total) : PlaceOrderSagaState
        {
            public ReservingInventory ReserveInventory() =>
                new ReservingInventory(SagaId, OrderId, CustomerId, Items, Total);
        }
        
        // Inventory reservation in progress
        public sealed record ReservingInventory(
            SagaId SagaId,
            OrderId OrderId,
            CustomerId CustomerId,
            IReadOnlyList<OrderLine> Items,
            Money Total) : PlaceOrderSagaState
        {
            public InventoryReserved OnReserved(ReservationId reservationId) =>
                new InventoryReserved(SagaId, OrderId, CustomerId, Items, Total, reservationId);
            
            public Failed OnReservationFailed(string reason) =>
                new Failed(SagaId, OrderId, reason, compensationSteps: new List<string>());
        }
        
        // Inventory reserved successfully
        public sealed record InventoryReserved(
            SagaId SagaId,
            OrderId OrderId,
            CustomerId CustomerId,
            IReadOnlyList<OrderLine> Items,
            Money Total,
            ReservationId ReservationId) : PlaceOrderSagaState
        {
            public ProcessingPayment ProcessPayment() =>
                new ProcessingPayment(SagaId, OrderId, CustomerId, Items, Total, ReservationId);
            
            public Compensating Compensate(string reason) =>
                new Compensating(
                    SagaId,
                    OrderId,
                    reason,
                    new List<CompensationStep>
                    {
                        new CompensationStep.ReleaseInventory(ReservationId)
                    });
        }
        
        // Payment processing in progress
        public sealed record ProcessingPayment(
            SagaId SagaId,
            OrderId OrderId,
            CustomerId CustomerId,
            IReadOnlyList<OrderLine> Items,
            Money Total,
            ReservationId ReservationId) : PlaceOrderSagaState
        {
            public PaymentProcessed OnPaymentProcessed(PaymentId paymentId) =>
                new PaymentProcessed(SagaId, OrderId, CustomerId, Items, Total, ReservationId, paymentId);
            
            public Compensating OnPaymentFailed(string reason) =>
                new Compensating(
                    SagaId,
                    OrderId,
                    reason,
                    new List<CompensationStep>
                    {
                        new CompensationStep.ReleaseInventory(ReservationId)
                    });
        }
        
        // Payment processed successfully
        public sealed record PaymentProcessed(
            SagaId SagaId,
            OrderId OrderId,
            CustomerId CustomerId,
            IReadOnlyList<OrderLine> Items,
            Money Total,
            ReservationId ReservationId,
            PaymentId PaymentId) : PlaceOrderSagaState
        {
            public CreatingShipment CreateShipment() =>
                new CreatingShipment(SagaId, OrderId, CustomerId, Items, Total, ReservationId, PaymentId);
            
            public Compensating Compensate(string reason) =>
                new Compensating(
                    SagaId,
                    OrderId,
                    reason,
                    new List<CompensationStep>
                    {
                        new CompensationStep.RefundPayment(PaymentId),
                        new CompensationStep.ReleaseInventory(ReservationId)
                    });
        }
        
        // Shipment creation in progress
        public sealed record CreatingShipment(
            SagaId SagaId,
            OrderId OrderId,
            CustomerId CustomerId,
            IReadOnlyList<OrderLine> Items,
            Money Total,
            ReservationId ReservationId,
            PaymentId PaymentId) : PlaceOrderSagaState
        {
            public Completed OnShipmentCreated(ShipmentId shipmentId) =>
                new Completed(SagaId, OrderId, shipmentId);
            
            public Compensating OnShipmentFailed(string reason) =>
                new Compensating(
                    SagaId,
                    OrderId,
                    reason,
                    new List<CompensationStep>
                    {
                        new CompensationStep.RefundPayment(PaymentId),
                        new CompensationStep.ReleaseInventory(ReservationId)
                    });
        }
        
        // Compensating transaction in progress
        public sealed record Compensating(
            SagaId SagaId,
            OrderId OrderId,
            string Reason,
            IReadOnlyList<CompensationStep> Steps) : PlaceOrderSagaState
        {
            public Failed OnCompensationComplete() =>
                new Failed(
                    SagaId,
                    OrderId,
                    Reason,
                    Steps.Select(s => s.ToString()).ToList());
        }
        
        // Saga completed successfully
        public sealed record Completed(
            SagaId SagaId,
            OrderId OrderId,
            ShipmentId ShipmentId) : PlaceOrderSagaState;
        
        // Saga failed and compensated
        public sealed record Failed(
            SagaId SagaId,
            OrderId OrderId,
            string Reason,
            IReadOnlyList<string> CompensationSteps) : PlaceOrderSagaState;
    }
    
    /// <summary>
    /// Compensation steps as types.
    /// </summary>
    public abstract record CompensationStep
    {
        private CompensationStep() { }
        
        public sealed record ReleaseInventory(ReservationId ReservationId) : CompensationStep;
        public sealed record RefundPayment(PaymentId PaymentId) : CompensationStep;
    }
    
    /// <summary>
    /// Saga orchestrator executes steps and handles compensation.
    /// </summary>
    public sealed class PlaceOrderSaga
    {
        private readonly IInventoryService _inventory;
        private readonly IPaymentService _payment;
        private readonly IShippingService _shipping;
        private readonly ISagaStateRepository _stateRepo;
        
        public PlaceOrderSaga(
            IInventoryService inventory,
            IPaymentService payment,
            IShippingService shipping,
            ISagaStateRepository stateRepo)
        {
            _inventory = inventory;
            _payment = payment;
            _shipping = shipping;
            _stateRepo = stateRepo;
        }
        
        public async Task<Result<PlaceOrderSagaState, string>> Execute(
            PlaceOrderSagaState.Started state)
        {
            var currentState = (PlaceOrderSagaState)state;
            
            try
            {
                // Step 1: Reserve inventory
                currentState = state.ReserveInventory();
                await _stateRepo.SaveAsync(currentState);
                
                var reserveResult = await _inventory.ReserveAsync(state.Items);
                
                if (!reserveResult.IsSuccess)
                {
                    var failedState = ((PlaceOrderSagaState.ReservingInventory)currentState)
                        .OnReservationFailed(reserveResult.Error);
                    await _stateRepo.SaveAsync(failedState);
                    return Result<PlaceOrderSagaState, string>.Success(failedState);
                }
                
                currentState = ((PlaceOrderSagaState.ReservingInventory)currentState)
                    .OnReserved(reserveResult.Value);
                await _stateRepo.SaveAsync(currentState);
                
                // Step 2: Process payment
                currentState = ((PlaceOrderSagaState.InventoryReserved)currentState)
                    .ProcessPayment();
                await _stateRepo.SaveAsync(currentState);
                
                var paymentResult = await _payment.ChargeAsync(state.CustomerId, state.Total);
                
                if (!paymentResult.IsSuccess)
                {
                    // Compensate: Release inventory
                    var compensatingState = ((PlaceOrderSagaState.ProcessingPayment)currentState)
                        .OnPaymentFailed(paymentResult.Error);
                    await _stateRepo.SaveAsync(compensatingState);
                    await Compensate(compensatingState);
                    return Result<PlaceOrderSagaState, string>.Success(compensatingState);
                }
                
                currentState = ((PlaceOrderSagaState.ProcessingPayment)currentState)
                    .OnPaymentProcessed(paymentResult.Value);
                await _stateRepo.SaveAsync(currentState);
                
                // Step 3: Create shipment
                currentState = ((PlaceOrderSagaState.PaymentProcessed)currentState)
                    .CreateShipment();
                await _stateRepo.SaveAsync(currentState);
                
                var shipmentResult = await _shipping.CreateShipmentAsync(state.OrderId);
                
                if (!shipmentResult.IsSuccess)
                {
                    // Compensate: Refund payment and release inventory
                    var compensatingState = ((PlaceOrderSagaState.CreatingShipment)currentState)
                        .OnShipmentFailed(shipmentResult.Error);
                    await _stateRepo.SaveAsync(compensatingState);
                    await Compensate(compensatingState);
                    return Result<PlaceOrderSagaState, string>.Success(compensatingState);
                }
                
                currentState = ((PlaceOrderSagaState.CreatingShipment)currentState)
                    .OnShipmentCreated(shipmentResult.Value);
                await _stateRepo.SaveAsync(currentState);
                
                return Result<PlaceOrderSagaState, string>.Success(currentState);
            }
            catch (Exception ex)
            {
                // On any exception, attempt compensation
                var failedState = new PlaceOrderSagaState.Failed(
                    state.SagaId,
                    state.OrderId,
                    ex.Message,
                    new List<string> { "Exception during saga execution" });
                
                await _stateRepo.SaveAsync(failedState);
                return Result<PlaceOrderSagaState, string>.Failure(ex.Message);
            }
        }
        
        private async Task Compensate(PlaceOrderSagaState.Compensating state)
        {
            foreach (var step in state.Steps.Reverse())
            {
                try
                {
                    await step switch
                    {
                        CompensationStep.ReleaseInventory release =>
                            _inventory.ReleaseReservationAsync(release.ReservationId),
                        
                        CompensationStep.RefundPayment refund =>
                            _payment.RefundAsync(refund.PaymentId),
                        
                        _ => throw new InvalidOperationException("Unknown compensation step")
                    };
                }
                catch (Exception ex)
                {
                    // Log compensation failure but continue with remaining steps
                    // In production, might want dead letter queue for manual intervention
                }
            }
            
            var failedState = state.OnCompensationComplete();
            await _stateRepo.SaveAsync(failedState);
        }
    }
}
```

## Saga Event-Driven Approach

```csharp
// Alternative: Event-driven saga (choreography)
public abstract record SagaEvent
{
    public sealed record OrderCreated(
        OrderId OrderId,
        CustomerId CustomerId,
        Money Total) : SagaEvent;
    
    public sealed record InventoryReserved(
        OrderId OrderId,
        ReservationId ReservationId) : SagaEvent;
    
    public sealed record InventoryReservationFailed(
        OrderId OrderId,
        string Reason) : SagaEvent;
    
    public sealed record PaymentProcessed(
        OrderId OrderId,
        PaymentId PaymentId) : SagaEvent;
    
    public sealed record PaymentFailed(
        OrderId OrderId,
        string Reason) : SagaEvent;
    
    public sealed record ShipmentCreated(
        OrderId OrderId,
        ShipmentId ShipmentId) : SagaEvent;
}

// Each service handles its events and publishes next event
public sealed class InventoryEventHandler : IEventHandler<SagaEvent.OrderCreated>
{
    private readonly IInventoryService _inventory;
    private readonly IEventPublisher _publisher;
    
    public async Task HandleAsync(SagaEvent.OrderCreated evt)
    {
        var result = await _inventory.ReserveAsync(evt.OrderId);
        
        if (result.IsSuccess)
        {
            await _publisher.PublishAsync(new SagaEvent.InventoryReserved(
                evt.OrderId,
                result.Value));
        }
        else
        {
            await _publisher.PublishAsync(new SagaEvent.InventoryReservationFailed(
                evt.OrderId,
                result.Error));
        }
    }
}
```

## Saga Persistence

```csharp
public interface ISagaStateRepository
{
    Task<Option<PlaceOrderSagaState>> GetAsync(SagaId id);
    Task SaveAsync(PlaceOrderSagaState state);
}

// Persist saga state for recovery after failures
public class SagaStateRepository : ISagaStateRepository
{
    private readonly DbContext _db;
    
    public async Task SaveAsync(PlaceOrderSagaState state)
    {
        var entity = new SagaStateEntity
        {
            Id = state switch
            {
                PlaceOrderSagaState.Started s => s.SagaId.Value,
                PlaceOrderSagaState.ReservingInventory s => s.SagaId.Value,
                PlaceOrderSagaState.InventoryReserved s => s.SagaId.Value,
                PlaceOrderSagaState.ProcessingPayment s => s.SagaId.Value,
                PlaceOrderSagaState.PaymentProcessed s => s.SagaId.Value,
                PlaceOrderSagaState.CreatingShipment s => s.SagaId.Value,
                PlaceOrderSagaState.Compensating s => s.SagaId.Value,
                PlaceOrderSagaState.Completed s => s.SagaId.Value,
                PlaceOrderSagaState.Failed s => s.SagaId.Value,
                _ => throw new InvalidOperationException("Unknown state")
            },
            StateType = state.GetType().Name,
            StateData = JsonSerializer.Serialize(state),
            UpdatedAt = DateTimeOffset.UtcNow
        };
        
        _db.SagaStates.Update(entity);
        await _db.SaveChangesAsync();
    }
}
```

## Why It's a Problem

1. **Inconsistent state**: Failed operations leave partial changes
2. **Manual compensation**: Error-prone rollback logic
3. **Lost progress**: Can't resume after crashes
4. **Hidden state**: Difficult to track workflow progress

## Symptoms

- Try-catch blocks attempting manual rollback
- Inconsistent data across services after failures
- No way to resume operations after crashes
- Difficulty tracking multi-step workflows

## Benefits

- **Consistency**: Guaranteed compensation on failures
- **Resumability**: Can continue after crashes
- **Visibility**: Clear workflow state tracking
- **Type safety**: Compiler enforces valid transitions

## See Also

- [Type-Safe State Transitions](./type-safe-state-transitions.md) — state as types
- [Domain Events](./domain-events.md) — event-driven communication
- [Type-Safe Workflow Modeling](./type-safe-workflow-modeling.md) — modeling workflows
- [Discriminated Unions](./discriminated-unions.md) — modeling alternatives
