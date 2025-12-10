# Exhaustive Pattern Matching (Compiler-Enforced Completeness)

> Using if-else chains or switch with default that hide missing cases—use sealed type hierarchies and pattern matching to make incomplete handling a compile error.

## Problem

When modeling variants (payment types, document states, error cases), traditional approaches use enums or base classes with `default` cases. This allows you to forget handling new variants when they're added, causing runtime failures instead of compile errors.

## Example

### ❌ Before

```csharp
public enum PaymentMethod
{
    CreditCard,
    BankTransfer,
    Cash,
    Cryptocurrency  // Added later—compiler doesn't force updates
}

public class PaymentProcessor
{
    public decimal CalculateFee(PaymentMethod method, decimal amount)
    {
        switch (method)
        {
            case PaymentMethod.CreditCard:
                return amount * 0.029m;
            case PaymentMethod.BankTransfer:
                return 5.00m;
            default:
                return 0m;  // Crypto and Cash silently return 0!
        }
    }
    
    public bool RequiresVerification(PaymentMethod method)
    {
        // Forgot to update this method when adding Cryptocurrency
        if (method == PaymentMethod.CreditCard)
            return true;
        if (method == PaymentMethod.BankTransfer)
            return true;
        
        return false;  // Crypto gets no verification!
    }
}
```

**Problems:**
- Adding `Cryptocurrency` didn't break compilation
- `default` case hides missing implementations
- Multiple switch statements must all be updated
- No compiler help finding all places to update

### ✅ After (Sealed Hierarchy + Exhaustive Matching)

```csharp
/// <summary>
/// Base payment method. Sealed hierarchy ensures exhaustiveness.
/// </summary>
public abstract record PaymentMethod
{
    // Private constructor prevents inheritance outside this file
    private PaymentMethod() { }
    
    /// <summary>
    /// All variants must be defined as nested types.
    /// </summary>
    public sealed record CreditCard(
        string CardNumber,
        string Cvv,
        DateTime Expiry) : PaymentMethod;
    
    public sealed record BankTransfer(
        string AccountNumber,
        string RoutingNumber) : PaymentMethod;
    
    public sealed record Cash() : PaymentMethod;
    
    public sealed record Cryptocurrency(
        string WalletAddress,
        string CurrencyCode) : PaymentMethod;
}

public class PaymentProcessor
{
    public decimal CalculateFee(PaymentMethod method, decimal amount)
    {
        // No default case—compiler forces exhaustive matching
        return method switch
        {
            PaymentMethod.CreditCard => amount * 0.029m,
            PaymentMethod.BankTransfer => 5.00m,
            PaymentMethod.Cash => 0m,
            PaymentMethod.Cryptocurrency crypto => CalculateCryptoFee(crypto, amount)
            // If you comment out any case, compiler error CS8509
        };
    }
    
    decimal CalculateCryptoFee(PaymentMethod.Cryptocurrency crypto, decimal amount)
    {
        // Crypto-specific fee logic
        return crypto.CurrencyCode switch
        {
            "BTC" => 0.001m,
            "ETH" => 0.01m,
            _ => 0.005m
        };
    }
    
    public bool RequiresVerification(PaymentMethod method)
    {
        // Compiler forces all cases
        return method switch
        {
            PaymentMethod.CreditCard => true,
            PaymentMethod.BankTransfer => true,
            PaymentMethod.Cash => false,
            PaymentMethod.Cryptocurrency => true  // Can't forget this!
        };
    }
}

// Usage
var payment = new PaymentMethod.CreditCard("4111111111111111", "123", DateTime.Now.AddYears(2));
var fee = processor.CalculateFee(payment, 100m);
```

## Document States with Exhaustive Matching

```csharp
/// <summary>
/// Document lifecycle states as sealed hierarchy.
/// </summary>
public abstract record DocumentState
{
    private DocumentState() { }
    
    public sealed record Draft(
        DateTime CreatedAt,
        UserId CreatedBy) : DocumentState;
    
    public sealed record UnderReview(
        DateTime SubmittedAt,
        UserId SubmittedBy,
        UserId ReviewerId) : DocumentState;
    
    public sealed record Approved(
        DateTime ApprovedAt,
        UserId ApprovedBy,
        string ApprovalNotes) : DocumentState;
    
    public sealed record Rejected(
        DateTime RejectedAt,
        UserId RejectedBy,
        string RejectionReason) : DocumentState;
    
    public sealed record Published(
        DateTime PublishedAt,
        string PublishedUrl) : DocumentState;
}

public class Document
{
    public DocumentId Id { get; }
    public string Content { get; }
    public DocumentState State { get; private set; }
    
    public Document(DocumentId id, string content, UserId createdBy)
    {
        Id = id;
        Content = content;
        State = new DocumentState.Draft(DateTime.UtcNow, createdBy);
    }
    
    public Result<Unit, string> SubmitForReview(UserId reviewerId)
    {
        // Exhaustive matching on current state
        return State switch
        {
            DocumentState.Draft draft =>
                SubmitDraft(draft, reviewerId),
            
            DocumentState.UnderReview =>
                Result<Unit, string>.Failure("Already under review"),
            
            DocumentState.Approved =>
                Result<Unit, string>.Failure("Already approved, cannot resubmit"),
            
            DocumentState.Rejected rejected =>
                SubmitAfterRejection(rejected, reviewerId),
            
            DocumentState.Published =>
                Result<Unit, string>.Failure("Already published, cannot modify")
            
            // Compiler error if any state is missing
        };
    }
    
    Result<Unit, string> SubmitDraft(DocumentState.Draft draft, UserId reviewerId)
    {
        State = new DocumentState.UnderReview(
            DateTime.UtcNow,
            draft.CreatedBy,
            reviewerId);
        
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    Result<Unit, string> SubmitAfterRejection(
        DocumentState.Rejected rejected,
        UserId reviewerId)
    {
        State = new DocumentState.UnderReview(
            DateTime.UtcNow,
            rejected.RejectedBy,
            reviewerId);
        
        return Result<Unit, string>.Success(Unit.Value);
    }
    
    public bool CanEdit()
    {
        // Exhaustive match makes logic explicit
        return State switch
        {
            DocumentState.Draft => true,
            DocumentState.UnderReview => false,
            DocumentState.Approved => false,
            DocumentState.Rejected => true,
            DocumentState.Published => false
        };
    }
}
```

## Error Cases with Exhaustive Matching

```csharp
/// <summary>
/// All possible errors when processing an order.
/// </summary>
public abstract record OrderError
{
    private OrderError() { }
    
    public sealed record OrderNotFound(OrderId OrderId) : OrderError;
    
    public sealed record InvalidOrderState(
        OrderId OrderId,
        string CurrentState,
        string RequiredState) : OrderError;
    
    public sealed record InsufficientInventory(
        OrderId OrderId,
        ProductId ProductId,
        int Available,
        int Requested) : OrderError;
    
    public sealed record PaymentFailed(
        OrderId OrderId,
        string PaymentProvider,
        string ErrorCode,
        string ErrorMessage) : OrderError;
    
    public sealed record ShippingAddressInvalid(
        OrderId OrderId,
        string Reason) : OrderError;
}

public class OrderService
{
    public Result<Order, OrderError> ProcessOrder(OrderId orderId)
    {
        var orderResult = _repository.GetById(orderId);
        
        if (!orderResult.HasValue)
            return Result<Order, OrderError>.Failure(
                new OrderError.OrderNotFound(orderId));
        
        var order = orderResult.Value;
        
        // ... processing logic that returns Result<Order, OrderError>
        return Result<Order, OrderError>.Success(order);
    }
    
    public IActionResult HandleOrderError(OrderError error)
    {
        // Exhaustive matching forces handling all error types
        return error switch
        {
            OrderError.OrderNotFound notFound =>
                NotFound($"Order {notFound.OrderId} not found"),
            
            OrderError.InvalidOrderState invalidState =>
                BadRequest($"Order {invalidState.OrderId} is in {invalidState.CurrentState}, " +
                          $"required {invalidState.RequiredState}"),
            
            OrderError.InsufficientInventory inventory =>
                Conflict($"Product {inventory.ProductId} has {inventory.Available} available, " +
                        $"requested {inventory.Requested}"),
            
            OrderError.PaymentFailed payment =>
                PaymentRequired($"Payment failed: {payment.ErrorMessage}"),
            
            OrderError.ShippingAddressInvalid address =>
                BadRequest($"Invalid shipping address: {address.Reason}")
            
            // Compiler error CS8509 if any error type is missing
        };
    }
}
```

## API Response Types with Exhaustive Matching

```csharp
/// <summary>
/// All possible API responses for a resource operation.
/// </summary>
public abstract record ApiResponse<T>
{
    private ApiResponse() { }
    
    public sealed record Success(T Data) : ApiResponse<T>;
    
    public sealed record NotFound(string ResourceId) : ApiResponse<T>;
    
    public sealed record Unauthorized(string Reason) : ApiResponse<T>;
    
    public sealed record ValidationFailed(
        Dictionary<string, string[]> Errors) : ApiResponse<T>;
    
    public sealed record ServerError(
        string Message,
        string? TraceId) : ApiResponse<T>;
}

public class UserController
{
    public IActionResult GetUser(string userId)
    {
        var response = _userService.GetUser(userId);
        
        // Exhaustive matching converts domain response to HTTP response
        return response switch
        {
            ApiResponse<User>.Success success =>
                Ok(success.Data),
            
            ApiResponse<User>.NotFound notFound =>
                NotFound(new { Message = $"User {notFound.ResourceId} not found" }),
            
            ApiResponse<User>.Unauthorized unauthorized =>
                Forbid(unauthorized.Reason),
            
            ApiResponse<User>.ValidationFailed validation =>
                BadRequest(new { Errors = validation.Errors }),
            
            ApiResponse<User>.ServerError error =>
                StatusCode(500, new
                {
                    Message = error.Message,
                    TraceId = error.TraceId
                })
        };
    }
}
```

## Workflow Steps with Exhaustive Matching

```csharp
/// <summary>
/// Loan application workflow steps.
/// </summary>
public abstract record LoanApplicationStep
{
    private LoanApplicationStep() { }
    
    public sealed record ApplicationSubmitted(
        DateTime SubmittedAt,
        decimal RequestedAmount) : LoanApplicationStep;
    
    public sealed record DocumentsRequested(
        DateTime RequestedAt,
        List<string> RequiredDocuments) : LoanApplicationStep;
    
    public sealed record DocumentsReceived(
        DateTime ReceivedAt,
        List<string> ReceivedDocuments) : LoanApplicationStep;
    
    public sealed record UnderReview(
        DateTime ReviewStartedAt,
        string ReviewerId) : LoanApplicationStep;
    
    public sealed record Approved(
        DateTime ApprovedAt,
        decimal ApprovedAmount,
        decimal InterestRate) : LoanApplicationStep;
    
    public sealed record Denied(
        DateTime DeniedAt,
        string Reason) : LoanApplicationStep;
}

public class LoanApplicationService
{
    public string GetStatusMessage(LoanApplicationStep step)
    {
        // Exhaustive matching ensures all steps have messages
        return step switch
        {
            LoanApplicationStep.ApplicationSubmitted submitted =>
                $"Application submitted on {submitted.SubmittedAt:d} " +
                $"for ${submitted.RequestedAmount:N2}",
            
            LoanApplicationStep.DocumentsRequested requested =>
                $"Please provide: {string.Join(", ", requested.RequiredDocuments)}",
            
            LoanApplicationStep.DocumentsReceived received =>
                $"Received {received.ReceivedDocuments.Count} documents " +
                $"on {received.ReceivedAt:d}",
            
            LoanApplicationStep.UnderReview review =>
                $"Under review by {review.ReviewerId} since {review.ReviewStartedAt:d}",
            
            LoanApplicationStep.Approved approved =>
                $"Approved ${approved.ApprovedAmount:N2} at {approved.InterestRate:P}",
            
            LoanApplicationStep.Denied denied =>
                $"Denied: {denied.Reason}"
        };
    }
    
    public bool CanCancel(LoanApplicationStep step)
    {
        return step switch
        {
            LoanApplicationStep.ApplicationSubmitted => true,
            LoanApplicationStep.DocumentsRequested => true,
            LoanApplicationStep.DocumentsReceived => true,
            LoanApplicationStep.UnderReview => false,
            LoanApplicationStep.Approved => false,
            LoanApplicationStep.Denied => false
        };
    }
}
```

## Enabling Exhaustiveness Warnings

Ensure your project treats missing pattern cases as errors:

```xml
<!-- In your .csproj -->
<PropertyGroup>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <!-- Or specifically for pattern matching: -->
    <WarningsAsErrors>CS8509;CS8524;CS8509</WarningsAsErrors>
</PropertyGroup>
```

Compiler warnings to enable:
- **CS8509**: Switch expression doesn't handle all possible values
- **CS8524**: Switch expression doesn't handle all possible inputs
- **CS8600-CS8629**: Nullable reference warnings

## Why It's a Problem

1. **Silent failures**: Missing cases return default values or fall through
2. **Maintenance hazards**: Adding variants doesn't break existing code
3. **Hidden bugs**: Runtime failures instead of compile errors
4. **Testing burden**: Must write tests for every missing case
5. **Refactoring danger**: Can't confidently rename or remove variants

## Symptoms

- Switch statements with `default` that returns a fallback value
- If-else chains that don't cover all cases
- Comments like "TODO: handle other cases"
- Runtime exceptions for unhandled cases
- Defensive coding with redundant checks

## Benefits

- **Compile-time completeness**: Cannot forget to handle a variant
- **Refactoring safety**: Compiler finds all affected code
- **Self-documenting**: All variants and their handling are explicit
- **Eliminates defaults**: No silent failures or fallback values
- **Type-driven design**: Variants guide implementation

## Trade-offs

- **Verbosity**: Must handle every case, even if trivial
- **Sealed hierarchies**: Can't extend variants outside the file
- **Learning curve**: Pattern matching syntax requires familiarity
- **Refactoring cost**: Adding variants breaks many places (but that's the point!)

## See Also

- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md) — converting enums to types
- [Typed Errors](./typed-errors.md) — error cases as sealed hierarchies
- [Honest Functions](./honest-functions.md) — explicit outcomes
- [Type-Safe Workflow Modeling](./type-safe-workflow-modeling.md) — workflows as types
