# Type-Safe Workflow Modeling (Business Processes as Types)

> Business workflows with runtime state checks allow invalid transitions—model each workflow step as a distinct type to make illegal transitions unrepresentable.

## Problem

When workflows are modeled with status enums and runtime checks, invalid state transitions can slip through. The type system can't prevent calling `Deliver()` before `Ship()`, or `Refund()` after `Deliver()`. Business processes should be encoded in types.

## Example

### ❌ Before

```csharp
public enum OrderStatus
{
    Draft,
    Submitted,
    Paid,
    Shipped,
    Delivered,
    Cancelled,
    Refunded
}

public class Order
{
    public OrderId Id { get; set; }
    public OrderStatus Status { get; set; }
    public List<OrderLine> Items { get; set; }
    public decimal Total { get; set; }
    
    // Optional fields that may or may not be present
    public DateTime? SubmittedAt { get; set; }
    public string? PaymentTransactionId { get; set; }
    public DateTime? PaidAt { get; set; }
    public string? TrackingNumber { get; set; }
    public DateTime? ShippedAt { get; set; }
    public DateTime? DeliveredAt { get; set; }
    public string? RefundTransactionId { get; set; }
    public DateTime? RefundedAt { get; set; }
    
    // Runtime checks everywhere
    public void Submit()
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Can only submit draft orders");
        
        if (Items.Count == 0)
            throw new InvalidOperationException("Cannot submit empty order");
        
        Status = OrderStatus.Submitted;
        SubmittedAt = DateTime.UtcNow;
    }
    
    public void Pay(string transactionId)
    {
        if (Status != OrderStatus.Submitted)
            throw new InvalidOperationException("Can only pay submitted orders");
        
        Status = OrderStatus.Paid;
        PaymentTransactionId = transactionId;
        PaidAt = DateTime.UtcNow;
    }
    
    public void Ship(string trackingNumber)
    {
        if (Status != OrderStatus.Paid)
            throw new InvalidOperationException("Can only ship paid orders");
        
        Status = OrderStatus.Shipped;
        TrackingNumber = trackingNumber;
        ShippedAt = DateTime.UtcNow;
    }
    
    public void Deliver()
    {
        if (Status != OrderStatus.Shipped)
            throw new InvalidOperationException("Can only deliver shipped orders");
        
        Status = OrderStatus.Delivered;
        DeliveredAt = DateTime.UtcNow;
    }
    
    public void Refund(string refundTransactionId)
    {
        // Complex validation: can refund paid or delivered orders
        if (Status != OrderStatus.Paid && Status != OrderStatus.Delivered)
            throw new InvalidOperationException("Can only refund paid or delivered orders");
        
        Status = OrderStatus.Refunded;
        RefundTransactionId = refundTransactionId;
        RefundedAt = DateTime.UtcNow;
    }
}

// Problems:
// - Can call methods in wrong order (compile succeeds, runtime fails)
// - Optional fields that should be required in certain states
// - No IDE autocomplete showing valid transitions
// - Cannot track which fields are valid in which states
```

### ✅ After

```csharp
/// <summary>
/// Order workflow modeled as distinct types.
/// Each state transition returns a new type, making invalid transitions unrepresentable.
/// </summary>
public abstract record OrderWorkflow
{
    public OrderId Id { get; init; }
    public CustomerId CustomerId { get; init; }
    public NonEmptyList<OrderLine> Items { get; init; }
    
    protected OrderWorkflow(OrderId id, CustomerId customerId, NonEmptyList<OrderLine> items)
    {
        Id = id;
        CustomerId = customerId;
        Items = items;
    }
}

/// <summary>
/// Draft: Order is being created, can add/remove items.
/// </summary>
public sealed record DraftOrder : OrderWorkflow
{
    private readonly List<OrderLine> _items;
    
    public new List<OrderLine> Items => _items;
    
    private DraftOrder(
        OrderId id,
        CustomerId customerId,
        List<OrderLine> items) 
        : base(id, customerId, NonEmptyList<OrderLine>.Create(items[0], items.Skip(1).ToArray()))
    {
        _items = items;
    }
    
    public static DraftOrder Create(OrderId id, CustomerId customerId)
    {
        return new DraftOrder(id, customerId, new List<OrderLine>());
    }
    
    public void AddItem(OrderLine item) => _items.Add(item);
    public void RemoveItem(ProductId productId) => _items.RemoveAll(i => i.ProductId == productId);
    
    /// <summary>
    /// Submit: Transition to SubmittedOrder.
    /// Compiler enforces: can only submit from Draft state.
    /// </summary>
    public Result<SubmittedOrder, string> Submit()
    {
        if (_items.Count == 0)
            return Result<SubmittedOrder, string>.Failure("Cannot submit empty order");
        
        var itemsList = NonEmptyList<OrderLine>.FromList(_items).Value;
        
        return Result<SubmittedOrder, string>.Success(
            new SubmittedOrder(Id, CustomerId, itemsList, DateTimeOffset.UtcNow));
    }
}

/// <summary>
/// Submitted: Order is awaiting payment.
/// Transition from Draft. Can proceed to Paid or be Cancelled.
/// </summary>
public sealed record SubmittedOrder : OrderWorkflow
{
    public DateTimeOffset SubmittedAt { get; }
    
    public SubmittedOrder(
        OrderId id,
        CustomerId customerId,
        NonEmptyList<OrderLine> items,
        DateTimeOffset submittedAt)
        : base(id, customerId, items)
    {
        SubmittedAt = submittedAt;
    }
    
    /// <summary>
    /// Pay: Transition to PaidOrder.
    /// Compiler enforces: can only pay from Submitted state.
    /// </summary>
    public PaidOrder Pay(PaymentTransactionId transactionId)
    {
        return new PaidOrder(
            Id,
            CustomerId,
            Items,
            SubmittedAt,
            transactionId,
            DateTimeOffset.UtcNow);
    }
    
    /// <summary>
    /// Cancel: Transition to CancelledOrder.
    /// Compiler enforces: can only cancel from Submitted state.
    /// </summary>
    public CancelledOrder Cancel(string reason)
    {
        return new CancelledOrder(Id, CustomerId, Items, reason, DateTimeOffset.UtcNow);
    }
}

/// <summary>
/// Paid: Payment received, ready to ship.
/// Transition from Submitted. Can proceed to Shipped or Refunded.
/// </summary>
public sealed record PaidOrder : OrderWorkflow
{
    public DateTimeOffset SubmittedAt { get; }
    public PaymentTransactionId PaymentTransactionId { get; }
    public DateTimeOffset PaidAt { get; }
    
    public PaidOrder(
        OrderId id,
        CustomerId customerId,
        NonEmptyList<OrderLine> items,
        DateTimeOffset submittedAt,
        PaymentTransactionId paymentTransactionId,
        DateTimeOffset paidAt)
        : base(id, customerId, items)
    {
        SubmittedAt = submittedAt;
        PaymentTransactionId = paymentTransactionId;
        PaidAt = paidAt;
    }
    
    /// <summary>
    /// Ship: Transition to ShippedOrder.
    /// Compiler enforces: can only ship from Paid state.
    /// </summary>
    public ShippedOrder Ship(TrackingNumber trackingNumber)
    {
        return new ShippedOrder(
            Id,
            CustomerId,
            Items,
            SubmittedAt,
            PaymentTransactionId,
            PaidAt,
            trackingNumber,
            DateTimeOffset.UtcNow);
    }
    
    /// <summary>
    /// Refund: Transition to RefundedOrder.
    /// Compiler enforces: can refund from Paid state.
    /// </summary>
    public RefundedOrder Refund(RefundTransactionId refundTransactionId, string reason)
    {
        return new RefundedOrder(
            Id,
            CustomerId,
            Items,
            PaymentTransactionId,
            refundTransactionId,
            reason,
            DateTimeOffset.UtcNow);
    }
}

/// <summary>
/// Shipped: Order in transit.
/// Transition from Paid. Can proceed to Delivered.
/// </summary>
public sealed record ShippedOrder : OrderWorkflow
{
    public DateTimeOffset SubmittedAt { get; }
    public PaymentTransactionId PaymentTransactionId { get; }
    public DateTimeOffset PaidAt { get; }
    public TrackingNumber TrackingNumber { get; }
    public DateTimeOffset ShippedAt { get; }
    
    public ShippedOrder(
        OrderId id,
        CustomerId customerId,
        NonEmptyList<OrderLine> items,
        DateTimeOffset submittedAt,
        PaymentTransactionId paymentTransactionId,
        DateTimeOffset paidAt,
        TrackingNumber trackingNumber,
        DateTimeOffset shippedAt)
        : base(id, customerId, items)
    {
        SubmittedAt = submittedAt;
        PaymentTransactionId = paymentTransactionId;
        PaidAt = paidAt;
        TrackingNumber = trackingNumber;
        ShippedAt = shippedAt;
    }
    
    /// <summary>
    /// Deliver: Transition to DeliveredOrder.
    /// Compiler enforces: can only deliver from Shipped state.
    /// </summary>
    public DeliveredOrder Deliver()
    {
        return new DeliveredOrder(
            Id,
            CustomerId,
            Items,
            SubmittedAt,
            PaymentTransactionId,
            PaidAt,
            TrackingNumber,
            ShippedAt,
            DateTimeOffset.UtcNow);
    }
}

/// <summary>
/// Delivered: Terminal state, order complete.
/// Transition from Shipped. Can be Refunded.
/// </summary>
public sealed record DeliveredOrder : OrderWorkflow
{
    public DateTimeOffset SubmittedAt { get; }
    public PaymentTransactionId PaymentTransactionId { get; }
    public DateTimeOffset PaidAt { get; }
    public TrackingNumber TrackingNumber { get; }
    public DateTimeOffset ShippedAt { get; }
    public DateTimeOffset DeliveredAt { get; }
    
    public DeliveredOrder(
        OrderId id,
        CustomerId customerId,
        NonEmptyList<OrderLine> items,
        DateTimeOffset submittedAt,
        PaymentTransactionId paymentTransactionId,
        DateTimeOffset paidAt,
        TrackingNumber trackingNumber,
        DateTimeOffset shippedAt,
        DateTimeOffset deliveredAt)
        : base(id, customerId, items)
    {
        SubmittedAt = submittedAt;
        PaymentTransactionId = paymentTransactionId;
        PaidAt = paidAt;
        TrackingNumber = trackingNumber;
        ShippedAt = shippedAt;
        DeliveredAt = deliveredAt;
    }
    
    /// <summary>
    /// Refund: Transition to RefundedOrder.
    /// Compiler enforces: can refund from Delivered state.
    /// </summary>
    public RefundedOrder Refund(RefundTransactionId refundTransactionId, string reason)
    {
        return new RefundedOrder(
            Id,
            CustomerId,
            Items,
            PaymentTransactionId,
            refundTransactionId,
            reason,
            DateTimeOffset.UtcNow);
    }
}

/// <summary>
/// Cancelled: Terminal state, order cancelled before payment.
/// Transition from Submitted.
/// </summary>
public sealed record CancelledOrder : OrderWorkflow
{
    public string Reason { get; }
    public DateTimeOffset CancelledAt { get; }
    
    public CancelledOrder(
        OrderId id,
        CustomerId customerId,
        NonEmptyList<OrderLine> items,
        string reason,
        DateTimeOffset cancelledAt)
        : base(id, customerId, items)
    {
        Reason = reason;
        CancelledAt = cancelledAt;
    }
}

/// <summary>
/// Refunded: Terminal state, payment returned to customer.
/// Transition from Paid or Delivered.
/// </summary>
public sealed record RefundedOrder : OrderWorkflow
{
    public PaymentTransactionId PaymentTransactionId { get; }
    public RefundTransactionId RefundTransactionId { get; }
    public string Reason { get; }
    public DateTimeOffset RefundedAt { get; }
    
    public RefundedOrder(
        OrderId id,
        CustomerId customerId,
        NonEmptyList<OrderLine> items,
        PaymentTransactionId paymentTransactionId,
        RefundTransactionId refundTransactionId,
        string reason,
        DateTimeOffset refundedAt)
        : base(id, customerId, items)
    {
        PaymentTransactionId = paymentTransactionId;
        RefundTransactionId = refundTransactionId;
        Reason = reason;
        RefundedAt = refundedAt;
    }
}

// Usage: Type system enforces valid transitions
var draft = DraftOrder.Create(OrderId.New(), customerId);
draft.AddItem(orderLine);

var submitResult = draft.Submit();  // Returns Result<SubmittedOrder, string>
var submitted = submitResult.Value;

var paid = submitted.Pay(transactionId);  // Returns PaidOrder

var shipped = paid.Ship(trackingNumber);  // Returns ShippedOrder

var delivered = shipped.Deliver();  // Returns DeliveredOrder

// ✗ Compile errors prevent invalid transitions:
// draft.Ship(trackingNumber);  // Won't compile—Draft doesn't have Ship()
// submitted.Deliver();  // Won't compile—Submitted doesn't have Deliver()
// delivered.Pay(transactionId);  // Won't compile—Delivered doesn't have Pay()
```

## Pattern Matching on Workflow States

```csharp
public static class OrderWorkflowExtensions
{
    public static string GetStatusDisplay(this OrderWorkflow order) => order switch
    {
        DraftOrder => "Draft",
        SubmittedOrder => "Awaiting Payment",
        PaidOrder => "Payment Received",
        ShippedOrder shipped => $"Shipped (Tracking: {shipped.TrackingNumber})",
        DeliveredOrder delivered => $"Delivered on {delivered.DeliveredAt:d}",
        CancelledOrder cancelled => $"Cancelled: {cancelled.Reason}",
        RefundedOrder refunded => $"Refunded: {refunded.Reason}",
        _ => throw new InvalidOperationException("Unknown order state")
    };
    
    public static bool CanBeCancelled(this OrderWorkflow order) => order switch
    {
        SubmittedOrder => true,  // Only submitted orders can be cancelled
        _ => false
    };
    
    public static bool CanBeRefunded(this OrderWorkflow order) => order switch
    {
        PaidOrder => true,
        DeliveredOrder => true,
        _ => false
    };
}
```

## Repository Pattern with Workflow Types

```csharp
/// <summary>
/// Repository works with base type but returns specific states.
/// </summary>
public interface IOrderRepository
{
    Task<Option<OrderWorkflow>> GetByIdAsync(OrderId id);
    Task SaveDraftAsync(DraftOrder order);
    Task SaveSubmittedAsync(SubmittedOrder order);
    Task SavePaidAsync(PaidOrder order);
    Task SaveShippedAsync(ShippedOrder order);
    Task SaveDeliveredAsync(DeliveredOrder order);
    Task SaveCancelledAsync(CancelledOrder order);
    Task SaveRefundedAsync(RefundedOrder order);
}

public class OrderService
{
    private readonly IOrderRepository _repository;
    
    public async Task<Result<PaidOrder, string>> ProcessPayment(
        OrderId orderId,
        PaymentTransactionId transactionId)
    {
        var orderOpt = await _repository.GetByIdAsync(orderId);
        
        return await orderOpt.Match(
            onSome: async order => order switch
            {
                // Type system ensures we handle correct state
                SubmittedOrder submitted =>
                {
                    var paid = submitted.Pay(transactionId);
                    await _repository.SavePaidAsync(paid);
                    return Result<PaidOrder, string>.Success(paid);
                },
                _ => Result<PaidOrder, string>.Failure(
                    "Can only pay submitted orders")
            },
            onNone: () => Task.FromResult(
                Result<PaidOrder, string>.Failure("Order not found"))
        );
    }
}
```

## Complex Workflow: Document Approval

```csharp
public abstract record DocumentWorkflow
{
    public DocumentId Id { get; init; }
    public UserId AuthorId { get; init; }
}

public sealed record DraftDocument : DocumentWorkflow
{
    public string Content { get; private set; }
    
    public void UpdateContent(string newContent) => Content = newContent;
    
    public Result<SubmittedDocument, string> Submit()
    {
        if (string.IsNullOrWhiteSpace(Content))
            return Result<SubmittedDocument, string>.Failure(
                "Cannot submit empty document");
        
        return Result<SubmittedDocument, string>.Success(
            new SubmittedDocument(Id, AuthorId, Content, DateTimeOffset.UtcNow));
    }
}

public sealed record SubmittedDocument : DocumentWorkflow
{
    public string Content { get; init; }
    public DateTimeOffset SubmittedAt { get; init; }
    
    public UnderReviewDocument AssignReviewer(UserId reviewerId)
    {
        return new UnderReviewDocument(
            Id, AuthorId, Content, SubmittedAt, reviewerId, DateTimeOffset.UtcNow);
    }
}

public sealed record UnderReviewDocument : DocumentWorkflow
{
    public string Content { get; init; }
    public DateTimeOffset SubmittedAt { get; init; }
    public UserId ReviewerId { get; init; }
    public DateTimeOffset ReviewStartedAt { get; init; }
    
    public ApprovedDocument Approve(string comments)
    {
        return new ApprovedDocument(
            Id, AuthorId, Content, ReviewerId, comments, DateTimeOffset.UtcNow);
    }
    
    public RejectedDocument Reject(string reason, NonEmptyList<string> feedbackItems)
    {
        return new RejectedDocument(
            Id, AuthorId, Content, ReviewerId, reason, feedbackItems, DateTimeOffset.UtcNow);
    }
}

public sealed record ApprovedDocument : DocumentWorkflow
{
    public string Content { get; init; }
    public UserId ReviewerId { get; init; }
    public string ReviewerComments { get; init; }
    public DateTimeOffset ApprovedAt { get; init; }
    
    public PublishedDocument Publish()
    {
        return new PublishedDocument(
            Id, AuthorId, Content, ApprovedAt, DateTimeOffset.UtcNow);
    }
}

public sealed record RejectedDocument : DocumentWorkflow
{
    public string Content { get; init; }
    public UserId ReviewerId { get; init; }
    public string RejectionReason { get; init; }
    public NonEmptyList<string> Feedback { get; init; }
    public DateTimeOffset RejectedAt { get; init; }
    
    public DraftDocument ReviseAndReopen(string updatedContent)
    {
        return new DraftDocument 
        { 
            Id = Id, 
            AuthorId = AuthorId,
            Content = updatedContent 
        };
    }
}

public sealed record PublishedDocument : DocumentWorkflow
{
    public string Content { get; init; }
    public DateTimeOffset ApprovedAt { get; init; }
    public DateTimeOffset PublishedAt { get; init; }
    
    public ArchivedDocument Archive(string reason)
    {
        return new ArchivedDocument(Id, AuthorId, Content, reason, DateTimeOffset.UtcNow);
    }
}

public sealed record ArchivedDocument : DocumentWorkflow
{
    public string Content { get; init; }
    public string ArchiveReason { get; init; }
    public DateTimeOffset ArchivedAt { get; init; }
}
```

## Benefits

- **Compile-time enforcement**: Invalid transitions don't compile
- **State-specific data**: Each state has exactly the data it needs
- **IDE support**: Autocomplete shows available transitions
- **Self-documenting**: Types reveal workflow structure
- **Exhaustive handling**: Pattern matching ensures all states covered
- **No runtime checks**: Type system prevents invalid states
- **Refactoring safety**: Changes to workflow cause compile errors

## Why It's a Problem

1. **Runtime failures**: Invalid transitions discovered at runtime
2. **Optional field chaos**: Nullable fields that should be required in some states
3. **No guidance**: IDE can't suggest valid transitions
4. **Maintenance burden**: Adding states requires updating all runtime checks
5. **Testing complexity**: Must test all invalid transition combinations

## Symptoms

- Methods throwing `InvalidOperationException` for wrong state
- Many nullable/optional fields on entities
- Status enum with runtime checks everywhere
- Comments explaining valid state transitions
- Complex validation logic in each method

## See Also

- [Enum State Machine](./enum-state-machine.md) — refactoring from enum-based states
- [Type-Level State Machines](./type-level-state-machines.md) — compile-time state tracking
- [Enforcing Call Order](./enforcing-call-order.md) — method sequencing with types
- [Phantom Types](./phantom-types.md) — tracking state in type parameters
- [Domain Events](./domain-events.md) — announcing state transitions
