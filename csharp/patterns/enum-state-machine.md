# Enum State Machine to Type-Safe States

> Enum-based state machines with fields valid only in certain states—model each state as a distinct type.

## Problem

When enums represent state machine states, associated data (like timestamps or IDs) often only makes sense in specific states. The compiler can't enforce this, so accessing invalid fields becomes a runtime bug waiting to happen.

## Example

### ❌ Before

```csharp
public class SubscriptionTierUpgradeStateMachine
{
    enum State
    {
        NotStarted,
        WaitingForUserToSpecifyUpgradeStartTime,
        WaitingForUpgradeWindow,
        RunningUpgrade,
        Complete
    }

    State currentState = State.NotStarted;
    DateTime allowedUpgradeStartTime;  // Only valid after WaitingForUserToSpecifyUpgradeStartTime!
    CustomerId customerId;
    SubscriptionTier newTier;

    public SubscriptionTierUpgradeStateMachine(CustomerId customerId, SubscriptionTier newTier)
    {
        this.customerId = customerId;
        this.newTier = newTier;
    }

    public void Advance(ExternalServiceProvider serviceProvider)
    {
        switch (currentState)
        {
            case State.NotStarted:
                ValidateCustomerCandidacyForNewTier();
                serviceProvider.SendCustomerRequestForUpgradeTime();
                currentState = State.WaitingForUserToSpecifyUpgradeStartTime;
                return;
            case State.WaitingForUserToSpecifyUpgradeStartTime:
                if (serviceProvider.TryRetrieveCustomerUpgradeTime(customerId, out allowedUpgradeStartTime))
                {
                    currentState = State.WaitingForUpgradeWindow;
                }
                return;
            case State.WaitingForUpgradeWindow:
                if (DateTime.Now >= allowedUpgradeStartTime)
                {
                    serviceProvider.InitiateUpgrade(customerId, newTier);
                    currentState = State.RunningUpgrade;
                }
                return;
            case State.RunningUpgrade:
                if (serviceProvider.GetUpgradeStatus(customerId) == UpgradeStatus.Complete)
                {
                    currentState = State.Complete;
                }
                return;
            default:
                throw new Exception($"Unrecognized state: {currentState}");
        }
    }
}

// Bug: allowedUpgradeStartTime is default(DateTime) until state transitions past WaitingForUserToSpecifyUpgradeStartTime
```

### ✅ After

```csharp
public abstract record UpgradeState(CustomerId CustomerId, SubscriptionTier NewTier)
{
    public abstract UpgradeState Advance(ExternalServiceProvider serviceProvider);
}

public sealed record NotStarted(CustomerId CustomerId, SubscriptionTier NewTier) 
    : UpgradeState(CustomerId, NewTier)
{
    public override UpgradeState Advance(ExternalServiceProvider serviceProvider)
    {
        ValidateCustomerCandidacyForNewTier();
        serviceProvider.SendCustomerRequestForUpgradeTime();
        return new WaitingForUserToSpecifyUpgradeStartTime(CustomerId, NewTier);
    }
}

public sealed record WaitingForUserToSpecifyUpgradeStartTime(CustomerId CustomerId, SubscriptionTier NewTier) 
    : UpgradeState(CustomerId, NewTier)
{
    public override UpgradeState Advance(ExternalServiceProvider serviceProvider)
    {
        if (serviceProvider.TryRetrieveCustomerUpgradeTime(CustomerId, out var upgradeTime))
        {
            return new WaitingForUpgradeWindow(CustomerId, NewTier, upgradeTime);
        }
        return this;
    }
}

public sealed record WaitingForUpgradeWindow(
    CustomerId CustomerId, 
    SubscriptionTier NewTier, 
    DateTime AllowedUpgradeStartTime)  // Only exists when it's valid!
    : UpgradeState(CustomerId, NewTier)
{
    public override UpgradeState Advance(ExternalServiceProvider serviceProvider)
    {
        if (DateTime.Now >= AllowedUpgradeStartTime)
        {
            serviceProvider.InitiateUpgrade(CustomerId, NewTier);
            return new RunningUpgrade(CustomerId, NewTier);
        }
        return this;
    }
}

public sealed record RunningUpgrade(CustomerId CustomerId, SubscriptionTier NewTier) 
    : UpgradeState(CustomerId, NewTier)
{
    public override UpgradeState Advance(ExternalServiceProvider serviceProvider)
    {
        if (serviceProvider.GetUpgradeStatus(CustomerId) == UpgradeStatus.Complete)
        {
            return new UpgradeComplete(CustomerId, NewTier);
        }
        return this;
    }
}

public sealed record UpgradeComplete(CustomerId CustomerId, SubscriptionTier NewTier) 
    : UpgradeState(CustomerId, NewTier)
{
    public override UpgradeState Advance(ExternalServiceProvider serviceProvider) => this;
}

// Usage
UpgradeState state = new NotStarted(customerId, newTier);
while (state is not UpgradeComplete)
{
    state = state.Advance(serviceProvider);
}
```

## Why It's a Problem

1. **Invalid field access**: `allowedUpgradeStartTime` is `default(DateTime)` until a specific state—accessing it earlier is a silent bug.

2. **State/data mismatch**: Nothing prevents the combination of `State.NotStarted` with a populated `allowedUpgradeStartTime`.

3. **Growing complexity**: As states and their associated data grow, the class becomes a bag of fields with implicit validity rules.

## Symptoms

- Enum states with additional fields that are only relevant for some states
- Methods that check state before allowing operations
- Fields that can have values that don't make sense for the current state
- Manual validation of state combinations throughout the codebase
- Switch statements that grow with each new state

## Benefits

- **Data lives with its state**: `AllowedUpgradeStartTime` only exists in `WaitingForUpgradeWindow`
- **Invalid states are unrepresentable**: Can't access data that doesn't exist yet
- **Self-documenting transitions**: Each state's `Advance` method shows valid next states
- **Exhaustive pattern matching**: Compiler warns if you miss a state
- **Easier testing**: Each state can be tested in isolation

## See Also

- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md) — modeling variants with types
- [Enforcing Call Order](./enforcing-call-order.md) — method sequencing
- [Flag Arguments](./flag-arguments.md) — polymorphism over booleans
- [Type-Safe Workflow Modeling](./type-safe-workflow-modeling.md) — comprehensive workflow patterns
- [Type-Level State Machines](./type-level-state-machines.md) — compile-time state tracking
