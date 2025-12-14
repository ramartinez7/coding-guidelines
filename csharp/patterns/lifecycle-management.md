# Lifecycle Management (Managing Entity State Transitions Over Time)

> Entities that can be modified at any time without lifecycle constraints lead to invalid states—model entity lifecycles explicitly to control valid transitions.

## Problem

Entities without lifecycle management can be modified inappropriately, like trying to ship a cancelled order or activate an already active account.

## Example

### ❌ Before

```csharp
public class Subscription
{
    public Guid Id { get; set; }
    public DateTime StartDate { get; set; }
    public DateTime? EndDate { get; set; }
    public bool IsActive { get; set; }
    public bool IsCancelled { get; set; }
    
    // Can call these at any time—no lifecycle control
    public void Activate() => IsActive = true;
    public void Cancel() => IsCancelled = true;
    public void Renew(DateTime newEndDate) => EndDate = newEndDate;
}

// All of these are problematic:
var sub = new Subscription();
sub.Cancel();  // Cancel before activating?
sub.Renew(DateTime.Now);  // Renew after cancelling?
sub.Activate();  // Activate after cancelling?
```

### ✅ After

```csharp
// Lifecycle as distinct types
public abstract record SubscriptionState
{
    private SubscriptionState() { }
    
    public sealed record Draft(
        SubscriptionId Id,
        CustomerId CustomerId,
        Plan Plan) : SubscriptionState
    {
        public Active Activate(DateTimeOffset startDate, DateTimeOffset endDate) =>
            new Active(Id, CustomerId, Plan, startDate, endDate);
        
        public Cancelled CancelDraft(string reason) =>
            new Cancelled(Id, CustomerId, reason, DateTimeOffset.UtcNow);
    }
    
    public sealed record Active(
        SubscriptionId Id,
        CustomerId CustomerId,
        Plan Plan,
        DateTimeOffset StartDate,
        DateTimeOffset EndDate) : SubscriptionState
    {
        public Active Renew(DateTimeOffset newEndDate) =>
            this with { EndDate = newEndDate };
        
        public Suspended Suspend(string reason) =>
            new Suspended(Id, CustomerId, Plan, StartDate, EndDate, reason, DateTimeOffset.UtcNow);
        
        public Cancelled Cancel(string reason) =>
            new Cancelled(Id, CustomerId, reason, DateTimeOffset.UtcNow);
        
        public Expired Expire() =>
            new Expired(Id, CustomerId, StartDate, EndDate);
    }
    
    public sealed record Suspended(
        SubscriptionId Id,
        CustomerId CustomerId,
        Plan Plan,
        DateTimeOffset StartDate,
        DateTimeOffset EndDate,
        string SuspensionReason,
        DateTimeOffset SuspendedAt) : SubscriptionState
    {
        public Active Resume() =>
            new Active(Id, CustomerId, Plan, StartDate, EndDate);
        
        public Cancelled Cancel(string reason) =>
            new Cancelled(Id, CustomerId, reason, DateTimeOffset.UtcNow);
    }
    
    public sealed record Expired(
        SubscriptionId Id,
        CustomerId CustomerId,
        DateTimeOffset StartDate,
        DateTimeOffset EndDate) : SubscriptionState
    {
        public Active Renew(DateTimeOffset newEndDate) =>
            new Active(Id, CustomerId, new Plan("Renewed"), DateTimeOffset.UtcNow, newEndDate);
    }
    
    public sealed record Cancelled(
        SubscriptionId Id,
        CustomerId CustomerId,
        string Reason,
        DateTimeOffset CancelledAt) : SubscriptionState;
}
```

## Repository Pattern with Lifecycle

```csharp
public interface ISubscriptionRepository
{
    Task<Option<SubscriptionState>> GetStateAsync(SubscriptionId id);
    Task SaveStateAsync(SubscriptionState state);
}

public sealed class SubscriptionService
{
    private readonly ISubscriptionRepository _repo;
    
    public async Task<Result<Unit, string>> ActivateSubscription(
        SubscriptionId id,
        DateTimeOffset startDate,
        DateTimeOffset endDate)
    {
        var stateOpt = await _repo.GetStateAsync(id);
        
        if (stateOpt.IsNone)
            return Result<Unit, string>.Failure("Subscription not found");
        
        // Type-safe: Only Draft has Activate method
        if (stateOpt.Value is not SubscriptionState.Draft draft)
            return Result<Unit, string>.Failure("Can only activate draft subscriptions");
        
        var activeState = draft.Activate(startDate, endDate);
        await _repo.SaveStateAsync(activeState);
        
        return Result<Unit, string>.Success(Unit.Value);
    }
}
```

## See Also

- [Type-Safe State Transitions](./type-safe-state-transitions.md)
- [Aggregate Roots](./aggregate-roots.md)
- [Domain Events](./domain-events.md)
