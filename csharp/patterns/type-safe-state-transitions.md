# Type-Safe State Transitions (Compiler-Enforced State Machines)

> Runtime checks for valid state transitions allow bugs to reach production—use types to make invalid transitions uncompilable.

## Problem

State machines implemented with enums and runtime validation allow invalid transitions to compile. Bugs slip through because the compiler can't verify transition rules.

## Example

### ❌ Before

```csharp
public enum OrderState
{
    Draft,
    Submitted,
    Paid,
    Shipped,
    Delivered,
    Cancelled
}

public class Order
{
    public OrderState State { get; set; }
    public decimal Total { get; set; }
    public string TrackingNumber { get; set; }
    
    public bool Submit()
    {
        if (State != OrderState.Draft)
            return false;  // Runtime check
        
        State = OrderState.Submitted;
        return true;
    }
    
    public bool Pay()
    {
        if (State != OrderState.Submitted)
            return false;  // Runtime check
        
        State = OrderState.Paid;
        return true;
    }
    
    public bool Ship(string trackingNumber)
    {
        if (State != OrderState.Paid)
            return false;  // Runtime check
        
        State = OrderState.Shipped;
        TrackingNumber = trackingNumber;
        return true;
    }
    
    // BUG: Forgot to check state!
    public void Cancel()
    {
        State = OrderState.Cancelled;  // Oops, can cancel after shipped!
    }
}

// All of these compile but are invalid:
var order = new Order { State = OrderState.Draft };
order.State = OrderState.Delivered;  // Skip all states!
order.TrackingNumber = "123";  // Set tracking on draft order!
order.Cancel();  // After marking delivered
```

**Problems:**
- Invalid transitions compile
- State can be set directly, bypassing validation
- Properties valid only in certain states (TrackingNumber)
- Compiler can't help catch mistakes

### ✅ After

```csharp
// Each state is a distinct type
public abstract record OrderState
{
    private OrderState() { }  // Sealed hierarchy
    
    // Draft state
    public sealed record Draft(
        OrderId Id,
        CustomerId CustomerId,
        List<OrderLine> Lines) : OrderState
    {
        public Result<Submitted, string> Submit()
        {
            if (Lines.Count == 0)
                return Result<Submitted, string>.Failure("Cannot submit empty order");
            
            var total = Lines.Sum(l => l.Price * l.Quantity);
            
            if (total < 10.00m)
                return Result<Submitted, string>.Failure("Minimum order is $10");
            
            return Result<Submitted, string>.Success(
                new Submitted(Id, CustomerId, Lines));
        }
        
        public Cancelled Cancel(string reason) =>
            new Cancelled(Id, CustomerId, reason);
    }
    
    // Submitted state
    public sealed record Submitted(
        OrderId Id,
        CustomerId CustomerId,
        IReadOnlyList<OrderLine> Lines) : OrderState
    {
        public Money Total => Lines
            .Select(l => l.LineTotal)
            .Aggregate(Money.Zero, (sum, total) => sum + total);
        
        public Paid Pay(PaymentId paymentId, DateTimeOffset paidAt) =>
            new Paid(Id, CustomerId, Lines, paymentId, paidAt);
        
        public Cancelled Cancel(string reason) =>
            new Cancelled(Id, CustomerId, reason);
    }
    
    // Paid state (can be shipped)
    public sealed record Paid(
        OrderId Id,
        CustomerId CustomerId,
        IReadOnlyList<OrderLine> Lines,
        PaymentId PaymentId,
        DateTimeOffset PaidAt) : OrderState
    {
        public Shipped Ship(TrackingNumber trackingNumber, DateTimeOffset shippedAt) =>
            new Shipped(Id, CustomerId, Lines, PaymentId, PaidAt, trackingNumber, shippedAt);
        
        // Cannot cancel after paid—must refund
        public Result<Cancelled, string> RequestRefund(string reason)
        {
            // Business rule: refund logic here
            return Result<Cancelled, string>.Success(
                new Cancelled(Id, CustomerId, reason));
        }
    }
    
    // Shipped state (has tracking number)
    public sealed record Shipped(
        OrderId Id,
        CustomerId CustomerId,
        IReadOnlyList<OrderLine> Lines,
        PaymentId PaymentId,
        DateTimeOffset PaidAt,
        TrackingNumber TrackingNumber,  // Only available in this state!
        DateTimeOffset ShippedAt) : OrderState
    {
        public Delivered Deliver(DateTimeOffset deliveredAt) =>
            new Delivered(Id, CustomerId, Lines, PaymentId, PaidAt, TrackingNumber, ShippedAt, deliveredAt);
    }
    
    // Delivered state (terminal state)
    public sealed record Delivered(
        OrderId Id,
        CustomerId CustomerId,
        IReadOnlyList<OrderLine> Lines,
        PaymentId PaymentId,
        DateTimeOffset PaidAt,
        TrackingNumber TrackingNumber,
        DateTimeOffset ShippedAt,
        DateTimeOffset DeliveredAt) : OrderState
    {
        // Terminal state—no transitions out
    }
    
    // Cancelled state (terminal state)
    public sealed record Cancelled(
        OrderId Id,
        CustomerId CustomerId,
        string Reason) : OrderState
    {
        // Terminal state—no transitions out
    }
}

// Usage: Invalid states unrepresentable
var draft = new OrderState.Draft(OrderId.New(), CustomerId.New(), new List<OrderLine>());

// ✓ Valid: Can submit draft
var submitResult = draft.Submit();

// ✓ Valid: Can cancel draft
var cancelled = draft.Cancel("Customer request");

// ✗ Invalid: These DON'T COMPILE!
// draft.Ship(trackingNumber);  // Draft doesn't have Ship method
// draft.TrackingNumber;  // Draft doesn't have TrackingNumber property

// Continue workflow
if (submitResult.IsSuccess)
{
    var submitted = submitResult.Value;
    
    // ✓ Valid: Can pay submitted order
    var paid = submitted.Pay(PaymentId.New(), DateTimeOffset.UtcNow);
    
    // ✓ Valid: Can ship paid order
    var shipped = paid.Ship(TrackingNumber.Generate(), DateTimeOffset.UtcNow);
    
    // ✓ Valid: Now we have tracking number
    Console.WriteLine($"Tracking: {shipped.TrackingNumber}");
    
    // ✗ Invalid: Can't pay already-paid order (doesn't compile)
    // paid.Pay(PaymentId.New(), DateTimeOffset.UtcNow);
}
```

## Pattern Matching for State Handling

```csharp
public sealed class OrderController
{
    public IActionResult GetOrder(OrderId id)
    {
        var orderState = _orderRepo.GetState(id);
        
        // Exhaustive pattern matching
        return orderState switch
        {
            OrderState.Draft draft => Ok(new
            {
                Status = "Draft",
                Id = draft.Id,
                ItemCount = draft.Lines.Count,
                CanSubmit = true,
                CanCancel = true
            }),
            
            OrderState.Submitted submitted => Ok(new
            {
                Status = "Submitted",
                Id = submitted.Id,
                Total = submitted.Total.Amount,
                CanPay = true,
                CanCancel = true
            }),
            
            OrderState.Paid paid => Ok(new
            {
                Status = "Paid",
                Id = paid.Id,
                PaymentId = paid.PaymentId,
                PaidAt = paid.PaidAt,
                CanShip = true
            }),
            
            OrderState.Shipped shipped => Ok(new
            {
                Status = "Shipped",
                Id = shipped.Id,
                TrackingNumber = shipped.TrackingNumber,
                ShippedAt = shipped.ShippedAt,
                CanDeliver = true
            }),
            
            OrderState.Delivered delivered => Ok(new
            {
                Status = "Delivered",
                Id = delivered.Id,
                DeliveredAt = delivered.DeliveredAt
            }),
            
            OrderState.Cancelled cancelled => Ok(new
            {
                Status = "Cancelled",
                Id = cancelled.Id,
                Reason = cancelled.Reason
            }),
            
            _ => throw new InvalidOperationException("Unknown state")
        };
    }
}
```

## State Machine with Events

```csharp
// State transitions return events
public abstract record OrderState
{
    public abstract record StateTransition
    {
        private StateTransition() { }
        
        public sealed record OrderSubmitted(OrderId OrderId, Money Total, DateTimeOffset SubmittedAt) : StateTransition;
        public sealed record OrderPaid(OrderId OrderId, PaymentId PaymentId, DateTimeOffset PaidAt) : StateTransition;
        public sealed record OrderShipped(OrderId OrderId, TrackingNumber TrackingNumber, DateTimeOffset ShippedAt) : StateTransition;
        public sealed record OrderDelivered(OrderId OrderId, DateTimeOffset DeliveredAt) : StateTransition;
        public sealed record OrderCancelled(OrderId OrderId, string Reason, DateTimeOffset CancelledAt) : StateTransition;
    }
    
    public sealed record Draft(OrderId Id, CustomerId CustomerId, List<OrderLine> Lines) : OrderState
    {
        public Result<(Submitted State, StateTransition.OrderSubmitted Event), string> Submit()
        {
            if (Lines.Count == 0)
                return Result<(Submitted, StateTransition.OrderSubmitted), string>.Failure(
                    "Cannot submit empty order");
            
            var newState = new Submitted(Id, CustomerId, Lines);
            var evt = new StateTransition.OrderSubmitted(
                Id,
                newState.Total,
                DateTimeOffset.UtcNow);
            
            return Result<(Submitted, StateTransition.OrderSubmitted), string>.Success(
                (newState, evt));
        }
    }
}

// Repository stores current state
public interface IOrderStateRepository
{
    Task<Option<OrderState>> GetStateAsync(OrderId id);
    Task SaveStateAsync(OrderState state);
}

// Service orchestrates transitions and publishes events
public sealed class OrderService
{
    private readonly IOrderStateRepository _repo;
    private readonly IEventPublisher _eventPublisher;
    
    public async Task<Result<Unit, string>> SubmitOrder(OrderId orderId)
    {
        var stateOpt = await _repo.GetStateAsync(orderId);
        
        if (stateOpt.IsNone)
            return Result<Unit, string>.Failure("Order not found");
        
        // Type-safe: Only Draft has Submit method
        if (stateOpt.Value is not OrderState.Draft draft)
            return Result<Unit, string>.Failure("Order cannot be submitted in current state");
        
        var result = draft.Submit();
        
        if (!result.IsSuccess)
            return Result<Unit, string>.Failure(result.Error);
        
        var (newState, evt) = result.Value;
        
        await _repo.SaveStateAsync(newState);
        await _eventPublisher.PublishAsync(evt);
        
        return Result<Unit, string>.Success(Unit.Value);
    }
}
```

## Complex State Machine Example

```csharp
// Document workflow with approval
public abstract record DocumentState
{
    private DocumentState() { }
    
    public sealed record Draft(
        DocumentId Id,
        AuthorId AuthorId,
        string Content) : DocumentState
    {
        public Result<PendingReview, string> SubmitForReview(ReviewerId reviewerId)
        {
            if (string.IsNullOrWhiteSpace(Content))
                return Result<PendingReview, string>.Failure("Cannot submit empty document");
            
            return Result<PendingReview, string>.Success(
                new PendingReview(Id, AuthorId, Content, reviewerId, DateTimeOffset.UtcNow));
        }
        
        public Draft UpdateContent(string newContent) =>
            this with { Content = newContent };
    }
    
    public sealed record PendingReview(
        DocumentId Id,
        AuthorId AuthorId,
        string Content,
        ReviewerId AssignedReviewer,
        DateTimeOffset SubmittedAt) : DocumentState
    {
        public UnderReview StartReview(DateTimeOffset startedAt) =>
            new UnderReview(Id, AuthorId, Content, AssignedReviewer, SubmittedAt, startedAt);
        
        public Draft ReturnToDraft(string reason) =>
            new Draft(Id, AuthorId, Content);
    }
    
    public sealed record UnderReview(
        DocumentId Id,
        AuthorId AuthorId,
        string Content,
        ReviewerId ReviewerId,
        DateTimeOffset SubmittedAt,
        DateTimeOffset ReviewStartedAt) : DocumentState
    {
        public Approved Approve(string comments, DateTimeOffset approvedAt) =>
            new Approved(Id, AuthorId, Content, ReviewerId, SubmittedAt, ReviewStartedAt, comments, approvedAt);
        
        public Rejected Reject(string reason, DateTimeOffset rejectedAt) =>
            new Rejected(Id, AuthorId, Content, ReviewerId, SubmittedAt, ReviewStartedAt, reason, rejectedAt);
        
        public ChangesRequested RequestChanges(string feedback, DateTimeOffset requestedAt) =>
            new ChangesRequested(Id, AuthorId, Content, ReviewerId, SubmittedAt, ReviewStartedAt, feedback, requestedAt);
    }
    
    public sealed record ChangesRequested(
        DocumentId Id,
        AuthorId AuthorId,
        string Content,
        ReviewerId ReviewerId,
        DateTimeOffset SubmittedAt,
        DateTimeOffset ReviewStartedAt,
        string Feedback,
        DateTimeOffset RequestedAt) : DocumentState
    {
        public Draft ReviseContent(string newContent) =>
            new Draft(Id, AuthorId, newContent);
    }
    
    public sealed record Approved(
        DocumentId Id,
        AuthorId AuthorId,
        string Content,
        ReviewerId ReviewerId,
        DateTimeOffset SubmittedAt,
        DateTimeOffset ReviewStartedAt,
        string ReviewComments,
        DateTimeOffset ApprovedAt) : DocumentState
    {
        public Published Publish(DateTimeOffset publishedAt) =>
            new Published(Id, AuthorId, Content, ReviewerId, SubmittedAt, ReviewStartedAt, ReviewComments, ApprovedAt, publishedAt);
    }
    
    public sealed record Rejected(
        DocumentId Id,
        AuthorId AuthorId,
        string Content,
        ReviewerId ReviewerId,
        DateTimeOffset SubmittedAt,
        DateTimeOffset ReviewStartedAt,
        string RejectionReason,
        DateTimeOffset RejectedAt) : DocumentState;
    
    public sealed record Published(
        DocumentId Id,
        AuthorId AuthorId,
        string Content,
        ReviewerId ReviewerId,
        DateTimeOffset SubmittedAt,
        DateTimeOffset ReviewStartedAt,
        string ReviewComments,
        DateTimeOffset ApprovedAt,
        DateTimeOffset PublishedAt) : DocumentState
    {
        public Archived Archive(DateTimeOffset archivedAt) =>
            new Archived(Id, AuthorId, Content, ReviewerId, SubmittedAt, ReviewStartedAt, ReviewComments, ApprovedAt, PublishedAt, archivedAt);
    }
    
    public sealed record Archived(
        DocumentId Id,
        AuthorId AuthorId,
        string Content,
        ReviewerId ReviewerId,
        DateTimeOffset SubmittedAt,
        DateTimeOffset ReviewStartedAt,
        string ReviewComments,
        DateTimeOffset ApprovedAt,
        DateTimeOffset PublishedAt,
        DateTimeOffset ArchivedAt) : DocumentState;
}
```

## Why It's a Problem

1. **Runtime errors**: Invalid transitions compile but fail at runtime
2. **Incomplete validation**: Easy to forget state checks
3. **Invalid data**: Properties accessible in wrong states
4. **Compiler can't help**: No type-level verification of transitions

## Symptoms

- State stored as enum with separate validation methods
- Boolean flags to track state (`isShipped`, `isPaid`)
- Properties valid only in certain states but always accessible
- Runtime checks for valid transitions scattered throughout code
- Unit tests to verify invalid transitions are rejected

## Benefits

- **Compile-time safety**: Invalid transitions don't compile
- **Exhaustive handling**: Pattern matching enforces all states handled
- **Data correctness**: State-specific data only available in that state
- **Self-documenting**: Type system reveals valid state machine
- **Refactoring confidence**: Compiler catches breaking changes

## See Also

- [Type-Level State Machines](./type-level-state-machines.md) — advanced state encoding
- [Phantom Types](./phantom-types.md) — compile-time state tracking
- [Discriminated Unions](./discriminated-unions.md) — modeling alternatives
- [Exhaustive Pattern Matching](./exhaustive-pattern-matching.md) — compiler-enforced completeness
- [Making Invalid States Unrepresentable](./making-invalid-states-unrepresentable.md) — core principle
