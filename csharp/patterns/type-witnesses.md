# Type Witnesses (Compile-Time Proofs)

> Runtime checks scattered throughout code—use type witnesses to prove conditions at compile time and eliminate redundant validation.

## Problem

Many operations require preconditions (non-null, validated, authorized). Traditional approaches check these conditions repeatedly at runtime, even when previous checks already proved them. Type witnesses are tokens that prove a condition holds, eliminating redundant checks.

## Example

### ❌ Before

```csharp
public class DocumentService
{
    void ProcessDocument(Document doc, User user)
    {
        // Check 1: Is user authenticated?
        if (!user.IsAuthenticated)
            throw new UnauthorizedException("User not authenticated");
        
        // Check 2: Can user access document?
        if (!CanUserAccessDocument(user, doc))
            throw new ForbiddenException("Access denied");
        
        // Check 3: Is document valid?
        if (!doc.IsValid())
            throw new ValidationException("Invalid document");
        
        // Finally, do the work
        PerformProcessing(doc, user);
    }
    
    void PerformProcessing(Document doc, User user)
    {
        // Should we check again here? We did already... but what if someone
        // calls this method directly? Better check again to be safe.
        if (!user.IsAuthenticated)
            throw new UnauthorizedException();
        
        if (!CanUserAccessDocument(user, doc))
            throw new ForbiddenException();
        
        // ... actual processing
    }
    
    void ArchiveDocument(Document doc, User user)
    {
        // Same checks repeated again!
        if (!user.IsAuthenticated)
            throw new UnauthorizedException();
        
        if (!CanUserAccessDocument(user, doc))
            throw new ForbiddenException();
        
        // Archive logic
    }
}
```

**Problems:**
- Checks repeated in every method
- No compile-time guarantee that checks were performed
- Easy to forget a check when adding new methods
- Runtime overhead from redundant validation

### ✅ After (With Type Witnesses)

```csharp
/// <summary>
/// Witness that user is authenticated.
/// Cannot be forged—only AuthenticationService can create it.
/// </summary>
public readonly record struct AuthenticatedUser(UserId UserId, string Name)
{
    // Private constructor: can't be created directly
    internal AuthenticatedUser(UserId userId, string name) : this()
    {
        UserId = userId;
        Name = name;
    }
}

/// <summary>
/// Witness that document access was authorized for this user.
/// Unforgeable token proving authorization was checked.
/// </summary>
public readonly record struct DocumentAccessGrant(
    DocumentId DocumentId,
    UserId UserId,
    AccessLevel Level)
{
    // Private constructor: can only be created by authorization service
    internal DocumentAccessGrant(
        DocumentId documentId,
        UserId userId,
        AccessLevel level) : this()
    {
        DocumentId = documentId;
        UserId = userId;
        Level = level;
    }
}

/// <summary>
/// Witness that document has been validated.
/// </summary>
public readonly record struct ValidatedDocument(
    DocumentId Id,
    string Content,
    DateTimeOffset ValidatedAt)
{
    internal ValidatedDocument(
        DocumentId id,
        string content,
        DateTimeOffset validatedAt) : this()
    {
        Id = id;
        Content = content;
        ValidatedAt = validatedAt;
    }
}

public class AuthenticationService
{
    public Result<AuthenticatedUser, AuthError> Authenticate(string username, string password)
    {
        // Verify credentials
        var userId = VerifyCredentials(username, password);
        
        if (userId is null)
            return Result<AuthenticatedUser, AuthError>.Failure(
                new AuthError("Invalid credentials"));
        
        // Create witness proving authentication
        return Result<AuthenticatedUser, AuthError>.Success(
            new AuthenticatedUser(userId.Value, username));
    }
}

public class AuthorizationService
{
    public Result<DocumentAccessGrant, AuthError> AuthorizeAccess(
        AuthenticatedUser user,
        DocumentId documentId,
        AccessLevel requiredLevel)
    {
        // Check permissions
        if (!HasAccess(user.UserId, documentId, requiredLevel))
            return Result<DocumentAccessGrant, AuthError>.Failure(
                new AuthError("Access denied"));
        
        // Create witness proving authorization
        return Result<DocumentAccessGrant, AuthError>.Success(
            new DocumentAccessGrant(documentId, user.UserId, requiredLevel));
    }
}

public class DocumentValidator
{
    public Result<ValidatedDocument, ValidationError> Validate(Document doc)
    {
        // Perform validation
        if (string.IsNullOrWhiteSpace(doc.Content))
            return Result<ValidatedDocument, ValidationError>.Failure(
                new ValidationError("Content is required"));
        
        if (doc.Content.Length > 10000)
            return Result<ValidatedDocument, ValidationError>.Failure(
                new ValidationError("Content too long"));
        
        // Create witness proving validation
        return Result<ValidatedDocument, ValidationError>.Success(
            new ValidatedDocument(doc.Id, doc.Content, DateTimeOffset.UtcNow));
    }
}

public class DocumentService
{
    // Type signature proves all preconditions
    void ProcessDocument(
        ValidatedDocument doc,
        DocumentAccessGrant access)
    {
        // No checks needed! Types prove:
        // 1. User is authenticated (access.UserId exists)
        // 2. User is authorized (DocumentAccessGrant exists)
        // 3. Document is valid (ValidatedDocument exists)
        // 4. Access is for THIS document (doc.Id == access.DocumentId checked at boundary)
        
        PerformProcessing(doc, access);
    }
    
    void PerformProcessing(
        ValidatedDocument doc,
        DocumentAccessGrant access)
    {
        // No redundant checks! Types guarantee preconditions.
        // If this method is called, all checks were already done.
        
        // ... actual processing
    }
    
    void ArchiveDocument(
        ValidatedDocument doc,
        DocumentAccessGrant access)
    {
        // Same—no checks needed
        
        // Archive logic
    }
}

// Usage: Checks happen once at the boundary
public class DocumentController
{
    async Task<IActionResult> ProcessDocument(
        int documentId,
        [FromServices] AuthenticationService authService,
        [FromServices] AuthorizationService authzService,
        [FromServices] DocumentValidator validator,
        [FromServices] DocumentService docService)
    {
        var docId = new DocumentId(documentId);
        
        // Step 1: Authenticate (get witness)
        var authResult = await authService.Authenticate(GetUsername(), GetPassword());
        
        if (!authResult.IsSuccess)
            return Unauthorized();
        
        var authenticatedUser = authResult.Value;
        
        // Step 2: Authorize (get witness)
        var authzResult = await authzService.AuthorizeAccess(
            authenticatedUser,
            docId,
            AccessLevel.Write);
        
        if (!authzResult.IsSuccess)
            return Forbidden();
        
        var accessGrant = authzResult.Value;
        
        // Step 3: Load and validate document (get witness)
        var doc = await LoadDocument(docId);
        var validationResult = validator.Validate(doc);
        
        if (!validationResult.IsSuccess)
            return BadRequest(validationResult.Error);
        
        var validatedDoc = validationResult.Value;
        
        // Step 4: Process (all witnesses collected)
        docService.ProcessDocument(validatedDoc, accessGrant);
        
        return Ok();
    }
}
```

## Witness for Ordering Constraints

```csharp
/// <summary>
/// Witness that collection is sorted.
/// </summary>
public readonly record struct SortedList<T>(IReadOnlyList<T> Items) where T : IComparable<T>
{
    private SortedList(IReadOnlyList<T> items) : this()
    {
        Items = items;
    }
    
    public static SortedList<T> FromSorted(IReadOnlyList<T> items)
    {
        // Verify sort order
        for (int i = 1; i < items.Count; i++)
        {
            if (items[i].CompareTo(items[i - 1]) < 0)
                throw new ArgumentException("List is not sorted");
        }
        
        return new SortedList<T>(items);
    }
    
    public static SortedList<T> FromUnsorted(IReadOnlyList<T> items)
    {
        var sorted = items.OrderBy(x => x).ToList();
        return new SortedList<T>(sorted);
    }
}

// Binary search only makes sense on sorted lists
public static class SortedListExtensions
{
    // Type signature proves list is sorted
    public static int BinarySearch<T>(this SortedList<T> sortedList, T value)
        where T : IComparable<T>
    {
        // No need to check if sorted—type guarantees it
        int left = 0;
        int right = sortedList.Items.Count - 1;
        
        while (left <= right)
        {
            int mid = left + (right - left) / 2;
            int comparison = sortedList.Items[mid].CompareTo(value);
            
            if (comparison == 0)
                return mid;
            
            if (comparison < 0)
                left = mid + 1;
            else
                right = mid - 1;
        }
        
        return -1;
    }
}
```

## Witness for Mutually Exclusive States

```csharp
/// <summary>
/// State machine with witnesses for each state.
/// </summary>
public abstract record OrderState;

public sealed record Draft() : OrderState;
public sealed record Submitted(DateTimeOffset SubmittedAt) : OrderState;
public sealed record Paid(PaymentId PaymentId, DateTimeOffset PaidAt) : OrderState;
public sealed record Shipped(TrackingNumber Tracking, DateTimeOffset ShippedAt) : OrderState;

public sealed record Order<TState>(
    OrderId Id,
    IReadOnlyList<OrderItem> Items,
    TState State) where TState : OrderState
{
    public static Order<Draft> CreateDraft(OrderId id, IReadOnlyList<OrderItem> items)
    {
        return new Order<Draft>(id, items, new Draft());
    }
}

public static class OrderExtensions
{
    // Can only submit draft orders
    public static Order<Submitted> Submit(this Order<Draft> order)
    {
        var submitted = new Submitted(DateTimeOffset.UtcNow);
        return new Order<Submitted>(order.Id, order.Items, submitted);
    }
    
    // Can only pay submitted orders
    public static Order<Paid> Pay(this Order<Submitted> order, PaymentId paymentId)
    {
        var paid = new Paid(paymentId, DateTimeOffset.UtcNow);
        return new Order<Paid>(order.Id, order.Items, paid);
    }
    
    // Can only ship paid orders
    public static Order<Shipped> Ship(this Order<Paid> order, TrackingNumber tracking)
    {
        var shipped = new Shipped(tracking, DateTimeOffset.UtcNow);
        return new Order<Shipped>(order.Id, order.Items, shipped);
    }
}

// Shipping service requires proof that order was paid
public class ShippingService
{
    // Type witness: Order<Paid> proves payment was processed
    public TrackingNumber Ship(Order<Paid> order)
    {
        // No payment check needed—type proves it
        var tracking = GenerateTrackingNumber();
        SendToWarehouse(order);
        return tracking;
    }
}
```

## Witness for Non-Null References

```csharp
/// <summary>
/// Witness that value is definitely not null.
/// </summary>
public readonly record struct NotNull<T>(T Value) where T : class
{
    private NotNull(T value) : this()
    {
        Value = value;
    }
    
    public static NotNull<T> From(T? value)
    {
        if (value is null)
            throw new ArgumentNullException(nameof(value));
        
        return new NotNull<T>(value);
    }
    
    public static NotNull<T>? TryFrom(T? value)
    {
        return value is not null ? new NotNull<T>(value) : null;
    }
    
    // Implicit conversion to T (safe—we know it's not null)
    public static implicit operator T(NotNull<T> notNull) => notNull.Value;
}

// Method signature proves parameter isn't null
void ProcessUser(NotNull<User> user)
{
    // No null check needed—type proves it
    Console.WriteLine(user.Value.Name);
}
```

## Witness Composition

```csharp
/// <summary>
/// Multiple witnesses combined into one.
/// </summary>
public readonly record struct ValidatedAndAuthorizedDocument(
    ValidatedDocument Document,
    DocumentAccessGrant Access)
{
    public static Result<ValidatedAndAuthorizedDocument, Error> Create(
        ValidatedDocument doc,
        DocumentAccessGrant access)
    {
        // Verify witnesses match
        if (doc.Id != access.DocumentId)
            return Result<ValidatedAndAuthorizedDocument, Error>.Failure(
                new Error("Document ID mismatch"));
        
        return Result<ValidatedAndAuthorizedDocument, Error>.Success(
            new ValidatedAndAuthorizedDocument(doc, access));
    }
}

// Single witness proving both conditions
void ProcessSecurely(ValidatedAndAuthorizedDocument doc)
{
    // Type proves BOTH validation AND authorization
    // No checks needed
}
```

## Why It's a Problem

1. **Redundant checks**: Same validation repeated in every method
2. **Forgotten checks**: Easy to forget a check in new code paths
3. **Runtime overhead**: Unnecessary validation on hot paths
4. **Unclear contracts**: Method signature doesn't show preconditions

## Symptoms

- Guard clauses at the start of every method
- Comments like "// Assumes X is valid"
- Copy-pasted validation logic
- Unit tests that verify exception throwing for invalid inputs
- Performance bottlenecks from repeated validation

## Benefits

- **Compile-time proofs**: Impossible to forget checks
- **No redundant validation**: Check once, trust the witness
- **Self-documenting**: Type signature shows what's proved
- **Performance**: Eliminate redundant runtime checks
- **Refactoring safety**: Can't call method without witness

## Trade-offs

- **More types**: Each proof needs a witness type
- **Boundary complexity**: Must collect witnesses at edges
- **Learning curve**: Team must understand witness pattern
- **Witness management**: Must thread witnesses through call chain

## See Also

- [Capability Security](./capability-security.md) — capabilities are authorization witnesses
- [Phantom Types](./phantom-types.md) — encoding state in type parameters
- [Static Factory Methods](./static-factory-methods.md) — controlled construction
- [Honest Functions](./honest-functions.md) — explicit outcomes in types
