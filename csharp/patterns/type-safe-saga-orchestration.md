# Type-Safe Saga Orchestration (Workflow Compensation)

> Saga patterns with string-based step names and runtime compensation logic—use typed saga steps to enforce compensation at compile time.

## Problem

Long-running distributed transactions using saga patterns rely on runtime step tracking and compensation logic. Steps can be skipped, compensation can fail silently, or the wrong compensation handler can be invoked. The compiler can't verify that each step has proper compensation.

## Example

### ❌ Before

```csharp
public class OrderSaga
{
    public async Task ExecuteAsync(Order order)
    {
        var completedSteps = new List<string>();
        
        try
        {
            await ReserveInventory(order);
            completedSteps.Add("ReserveInventory");
            
            await ChargeCustomer(order);
            completedSteps.Add("ChargeCustomer");
            
            await ShipOrder(order);
            completedSteps.Add("ShipOrder");
        }
        catch (Exception)
        {
            // Manual compensation—error prone
            if (completedSteps.Contains("ChargeCustomer"))
                await RefundCustomer(order);
            
            if (completedSteps.Contains("ReserveInventory"))
                await ReleaseInventory(order);
            
            throw;
        }
    }
}
```

### ✅ After

```csharp
public interface ISagaStep<TInput, TOutput>
{
    Task<TOutput> ExecuteAsync(TInput input);
    Task CompensateAsync(TOutput output);
}

public sealed record InventoryReserved(OrderId OrderId, ReservationId ReservationId);

public sealed class ReserveInventoryStep 
    : ISagaStep<Order, InventoryReserved>
{
    public async Task<InventoryReserved> ExecuteAsync(Order order)
    {
        var reservationId = await _inventory.ReserveAsync(order.Items);
        return new InventoryReserved(order.Id, reservationId);
    }
    
    public async Task CompensateAsync(InventoryReserved output)
    {
        await _inventory.ReleaseAsync(output.ReservationId);
    }
}

public sealed record PaymentCharged(OrderId OrderId, TransactionId TransactionId);

public sealed class ChargeCustomerStep 
    : ISagaStep<InventoryReserved, PaymentCharged>
{
    public async Task<PaymentCharged> ExecuteAsync(InventoryReserved input)
    {
        var transactionId = await _payment.ChargeAsync(input.OrderId);
        return new PaymentCharged(input.OrderId, transactionId);
    }
    
    public async Task CompensateAsync(PaymentCharged output)
    {
        await _payment.RefundAsync(output.TransactionId);
    }
}

public sealed class TypedSaga<TInput, TOutput>
{
    private readonly List<Func<Task>> _compensations = new();
    
    public async Task<Result<TOutput, string>> ExecuteAsync(
        TInput input,
        params ISagaStep<object, object>[] steps)
    {
        object current = input;
        
        try
        {
            foreach (var step in steps)
            {
                var output = await step.ExecuteAsync(current);
                _compensations.Add(() => step.CompensateAsync(output));
                current = output;
            }
            
            return Result<TOutput, string>.Success((TOutput)current);
        }
        catch (Exception ex)
        {
            // Automatic compensation in reverse order
            _compensations.Reverse();
            foreach (var compensate in _compensations)
            {
                await compensate();
            }
            
            return Result<TOutput, string>.Failure(ex.Message);
        }
    }
}
```

## See Also

- [Saga Pattern](./saga-pattern.md)
- [Type-Safe Workflow Modeling](./type-safe-workflow-modeling.md)
- [Type-Safe State Transitions](./type-safe-state-transitions.md)
