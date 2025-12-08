# Typed Errors (Making Failure Cases Explicit)

> String error messages lose type information and force string parsing—use discriminated unions to represent specific error cases with type safety.

## Problem

When error handling uses generic `Result<T, string>` or `Exception`, callers can't handle specific error cases programmatically. String messages must be parsed, error codes compared, or exception types caught. Typed errors make failure cases first-class domain concepts.

## Example

### ❌ Before

```csharp
public class OrderService
{
    // Generic string errors—no type information
    public Result<Order, string> PlaceOrder(PlaceOrderRequest request)
    {
        if (!_customerRepo.Exists(request.CustomerId))
            return Result<Order, string>.Failure("Customer not found");
        
        if (request.Items.Count == 0)
            return Result<Order, string>.Failure("Order must have at least one item");
        
        foreach (var item in request.Items)
        {
            if (!_productRepo.Exists(item.ProductId))
                return Result<Order, string>.Failure($"Product {item.ProductId} not found");
            
            if (item.Quantity <= 0)
                return Result<Order, string>.Failure("Quantity must be positive");
        }
        
        var customer = _customerRepo.GetById(request.CustomerId);
        if (!customer.IsActive)
            return Result<Order, string>.Failure("Customer account is not active");
        
        if (customer.CreditLimit < request.Total)
            return Result<Order, string>.Failure("Exceeds customer credit limit");
        
        // Success case
        return Result<Order, string>.Success(new Order(/* ... */));
    }
}

// Caller must parse strings to handle specific errors
var result = orderService.PlaceOrder(request);
if (!result.IsSuccess)
{
    // ❌ String parsing is fragile and error-prone
    if (result.Error.Contains("not found"))
    {
        return NotFound(result.Error);
    }
    else if (result.Error.Contains("credit limit"))
    {
        return BadRequest(result.Error);
    }
    else
    {
        return StatusCode(500, result.Error);
    }
}
```

**Problems:**
- No type information about error cases
- String parsing fragile and error-prone
- Can't pattern match on error types
- No IDE autocomplete for error handling
- Easy to miss error cases
- Difficult to add structured error data

### ✅ After

```csharp
/// <summary>
/// Discriminated union of all possible order placement errors.
/// Each error case is a distinct type with relevant data.
/// </summary>
public abstract record PlaceOrderError
{
    private PlaceOrderError() { }  // Sealed hierarchy
    
    public sealed record CustomerNotFound(CustomerId CustomerId) : PlaceOrderError;
    public sealed record CustomerNotActive(CustomerId CustomerId) : PlaceOrderError;
    public sealed record ProductNotFound(ProductId ProductId) : PlaceOrderError;
    public sealed record InvalidQuantity(ProductId ProductId, int Quantity) : PlaceOrderError;
    public sealed record EmptyOrder : PlaceOrderError;
    public sealed record CreditLimitExceeded(Money Limit, Money RequestedAmount) : PlaceOrderError;
    public sealed record InsufficientStock(ProductId ProductId, int Available, int Requested) : PlaceOrderError;
}

public class OrderService
{
    // Typed errors—specific failure cases
    public async Task<Result<Order, PlaceOrderError>> PlaceOrder(PlaceOrderRequest request)
    {
        // Validate customer exists and is active
        var customerOpt = await _customerRepo.GetByIdAsync(request.CustomerId);
        if (customerOpt.IsNone)
            return Result<Order, PlaceOrderError>.Failure(
                new PlaceOrderError.CustomerNotFound(request.CustomerId));
        
        var customer = customerOpt.Value;
        if (!customer.IsActive)
            return Result<Order, PlaceOrderError>.Failure(
                new PlaceOrderError.CustomerNotActive(request.CustomerId));
        
        // Validate order has items
        if (request.Items.Count == 0)
            return Result<Order, PlaceOrderError>.Failure(
                new PlaceOrderError.EmptyOrder());
        
        // Validate each item
        foreach (var item in request.Items)
        {
            if (item.Quantity <= 0)
                return Result<Order, PlaceOrderError>.Failure(
                    new PlaceOrderError.InvalidQuantity(item.ProductId, item.Quantity));
            
            var productOpt = await _productRepo.GetByIdAsync(item.ProductId);
            if (productOpt.IsNone)
                return Result<Order, PlaceOrderError>.Failure(
                    new PlaceOrderError.ProductNotFound(item.ProductId));
            
            var product = productOpt.Value;
            if (product.StockQuantity < item.Quantity)
                return Result<Order, PlaceOrderError>.Failure(
                    new PlaceOrderError.InsufficientStock(
                        item.ProductId,
                        product.StockQuantity,
                        item.Quantity));
        }
        
        // Validate credit limit
        if (customer.CreditLimit < request.Total)
            return Result<Order, PlaceOrderError>.Failure(
                new PlaceOrderError.CreditLimitExceeded(
                    customer.CreditLimit,
                    request.Total));
        
        // Success case
        var order = await CreateOrderAsync(request, customer);
        return Result<Order, PlaceOrderError>.Success(order);
    }
}

// API Controller: Pattern matching on error types
public class OrdersController : ControllerBase
{
    [HttpPost]
    public async Task<ActionResult<OrderDto>> PlaceOrder(PlaceOrderRequest request)
    {
        var result = await _orderService.PlaceOrder(request);
        
        return result.Match(
            onSuccess: order => Ok(MapToDto(order)),
            onFailure: error => error switch
            {
                PlaceOrderError.CustomerNotFound e =>
                    NotFound($"Customer {e.CustomerId} not found"),
                
                PlaceOrderError.CustomerNotActive e =>
                    BadRequest($"Customer {e.CustomerId} is not active"),
                
                PlaceOrderError.ProductNotFound e =>
                    NotFound($"Product {e.ProductId} not found"),
                
                PlaceOrderError.InvalidQuantity e =>
                    BadRequest($"Invalid quantity {e.Quantity} for product {e.ProductId}"),
                
                PlaceOrderError.EmptyOrder =>
                    BadRequest("Order must contain at least one item"),
                
                PlaceOrderError.CreditLimitExceeded e =>
                    BadRequest($"Order total {e.RequestedAmount} exceeds credit limit {e.Limit}"),
                
                PlaceOrderError.InsufficientStock e =>
                    Conflict($"Only {e.Available} units available for product {e.ProductId}, requested {e.Requested}"),
                
                _ => StatusCode(500, "Unknown error")
            }
        );
    }
}
```

## Pattern: Error Hierarchies

```csharp
/// <summary>
/// Base error type for authentication domain.
/// </summary>
public abstract record AuthenticationError
{
    private AuthenticationError() { }  // Sealed hierarchy
    
    // Credential errors
    public sealed record InvalidCredentials : AuthenticationError;
    public sealed record AccountLocked(DateTimeOffset UnlocksAt) : AuthenticationError;
    public sealed record PasswordExpired(DateTimeOffset ExpiredAt) : AuthenticationError;
    
    // Account errors
    public sealed record AccountNotFound(string Username) : AuthenticationError;
    public sealed record AccountNotActivated(EmailAddress Email) : AuthenticationError;
    public sealed record AccountSuspended(string Reason) : AuthenticationError;
    
    // MFA errors
    public sealed record MfaRequired(MfaMethod Method) : AuthenticationError;
    public sealed record InvalidMfaCode : AuthenticationError;
    public sealed record MfaCodeExpired : AuthenticationError;
}

public class AuthenticationService
{
    public async Task<Result<AuthenticatedUser, AuthenticationError>> Authenticate(
        Username username,
        Password password)
    {
        var userOpt = await _userRepo.GetByUsernameAsync(username);
        if (userOpt.IsNone)
            return Result<AuthenticatedUser, AuthenticationError>.Failure(
                new AuthenticationError.AccountNotFound(username.Value));
        
        var user = userOpt.Value;
        
        if (user.IsLocked)
            return Result<AuthenticatedUser, AuthenticationError>.Failure(
                new AuthenticationError.AccountLocked(user.LockedUntil));
        
        if (!await _passwordService.VerifyAsync(password, user.PasswordHash))
        {
            await _userRepo.RecordFailedLoginAsync(user.Id);
            return Result<AuthenticatedUser, AuthenticationError>.Failure(
                new AuthenticationError.InvalidCredentials());
        }
        
        if (user.IsPasswordExpired)
            return Result<AuthenticatedUser, AuthenticationError>.Failure(
                new AuthenticationError.PasswordExpired(user.PasswordExpiresAt));
        
        if (!user.IsActivated)
            return Result<AuthenticatedUser, AuthenticationError>.Failure(
                new AuthenticationError.AccountNotActivated(user.Email));
        
        if (user.IsSuspended)
            return Result<AuthenticatedUser, AuthenticationError>.Failure(
                new AuthenticationError.AccountSuspended(user.SuspensionReason));
        
        if (user.MfaEnabled)
            return Result<AuthenticatedUser, AuthenticationError>.Failure(
                new AuthenticationError.MfaRequired(user.MfaMethod));
        
        var authenticatedUser = await CreateAuthenticatedUserAsync(user);
        return Result<AuthenticatedUser, AuthenticationError>.Success(authenticatedUser);
    }
}
```

## Pattern: Error with Retry Information

```csharp
public abstract record PaymentError
{
    private PaymentError() { }
    
    // Permanent failures—don't retry
    public abstract record PermanentError : PaymentError;
    public sealed record InvalidCardNumber : PermanentError;
    public sealed record CardExpired : PermanentError;
    public sealed record InsufficientFunds : PermanentError;
    public sealed record CardDeclined(string Reason) : PermanentError;
    
    // Transient failures—can retry
    public abstract record TransientError(int RetryAfterSeconds) : PaymentError;
    public sealed record GatewayTimeout(int RetryAfterSeconds) : TransientError(RetryAfterSeconds);
    public sealed record RateLimitExceeded(int RetryAfterSeconds) : TransientError(RetryAfterSeconds);
    public sealed record TemporaryUnavailable(int RetryAfterSeconds) : TransientError(RetryAfterSeconds);
}

public class PaymentService
{
    public async Task<Result<PaymentReceipt, PaymentError>> ProcessPayment(
        PaymentRequest request,
        int attemptNumber = 1)
    {
        var result = await _paymentGateway.ChargeAsync(request);
        
        if (result.IsSuccess)
            return Result<PaymentReceipt, PaymentError>.Success(result.Value);
        
        var error = result.Error;
        
        // Retry on transient errors
        if (error is PaymentError.TransientError transient && attemptNumber < 3)
        {
            await Task.Delay(transient.RetryAfterSeconds * 1000);
            return await ProcessPayment(request, attemptNumber + 1);
        }
        
        return Result<PaymentReceipt, PaymentError>.Failure(error);
    }
}
```

## Pattern: Multiple Error Types

```csharp
/// <summary>
/// Validation errors—collect all problems at once.
/// </summary>
public sealed record ValidationErrors(NonEmptyList<ValidationError> Errors)
{
    public override string ToString() =>
        string.Join(", ", Errors.All.Select(e => e.Message));
}

public sealed record ValidationError(string Field, string Message);

/// <summary>
/// Business rule violation—single critical failure.
/// </summary>
public abstract record BusinessRuleViolation
{
    private BusinessRuleViolation() { }
    
    public sealed record DuplicateEmail(EmailAddress Email) : BusinessRuleViolation;
    public sealed record InvalidAgeRange(int Age, int Min, int Max) : BusinessRuleViolation;
    public sealed record PastDate(DateOnly Date) : BusinessRuleViolation;
}

/// <summary>
/// Infrastructure error—system-level failure.
/// </summary>
public abstract record InfrastructureError
{
    private InfrastructureError() { }
    
    public sealed record DatabaseUnavailable(Exception Exception) : InfrastructureError;
    public sealed record ExternalServiceTimeout(string ServiceName) : InfrastructureError;
    public sealed record ConfigurationMissing(string Key) : InfrastructureError;
}

// Method can fail for multiple reasons
public async Task<Result<User, CreateUserError>> CreateUser(CreateUserRequest request)
{
    // Validation errors
    var validationResult = await ValidateRequest(request);
    if (validationResult.IsFailure)
        return Result<User, CreateUserError>.Failure(
            new CreateUserError.ValidationFailed(validationResult.Error));
    
    // Business rule errors
    var existingUser = await _userRepo.GetByEmailAsync(request.Email);
    if (existingUser.IsSome)
        return Result<User, CreateUserError>.Failure(
            new CreateUserError.BusinessRuleViolation(
                new BusinessRuleViolation.DuplicateEmail(request.Email)));
    
    // Infrastructure errors
    try
    {
        var user = User.Create(request.Email, request.Age);
        await _userRepo.SaveAsync(user);
        return Result<User, CreateUserError>.Success(user);
    }
    catch (Exception ex)
    {
        return Result<User, CreateUserError>.Failure(
            new CreateUserError.InfrastructureFailed(
                new InfrastructureError.DatabaseUnavailable(ex)));
    }
}

public abstract record CreateUserError
{
    private CreateUserError() { }
    
    public sealed record ValidationFailed(ValidationErrors Errors) : CreateUserError;
    public sealed record BusinessRuleViolation(BusinessRuleViolation Rule) : CreateUserError;
    public sealed record InfrastructureFailed(InfrastructureError Error) : CreateUserError;
}
```

## Pattern: Error Context

```csharp
/// <summary>
/// Error with context about where it occurred.
/// </summary>
public sealed record ErrorContext<TError>(
    TError Error,
    string Operation,
    DateTimeOffset OccurredAt,
    Dictionary<string, string> Metadata)
{
    public static ErrorContext<TError> Create(
        TError error,
        string operation,
        params (string key, string value)[] metadata)
    {
        return new ErrorContext<TError>(
            error,
            operation,
            DateTimeOffset.UtcNow,
            metadata.ToDictionary(x => x.key, x => x.value));
    }
    
    public ErrorContext<TError> WithMetadata(string key, string value)
    {
        var newMetadata = new Dictionary<string, string>(Metadata)
        {
            [key] = value
        };
        return this with { Metadata = newMetadata };
    }
}

public class OrderService
{
    public async Task<Result<Order, ErrorContext<PlaceOrderError>>> PlaceOrder(
        PlaceOrderRequest request)
    {
        var result = await PlaceOrderInternal(request);
        
        return result.Match(
            onSuccess: order => Result<Order, ErrorContext<PlaceOrderError>>.Success(order),
            onFailure: error => Result<Order, ErrorContext<PlaceOrderError>>.Failure(
                ErrorContext<PlaceOrderError>.Create(
                    error,
                    "PlaceOrder",
                    ("CustomerId", request.CustomerId.ToString()),
                    ("ItemCount", request.Items.Count.ToString()),
                    ("TotalAmount", request.Total.ToString())
                )
            )
        );
    }
}
```

## Benefits

- **Type safety**: Compiler ensures all error cases handled
- **Pattern matching**: Exhaustive error handling with switch expressions
- **IDE support**: Autocomplete shows all error cases
- **Structured data**: Errors carry relevant context
- **Refactoring safety**: Adding error cases causes compile errors
- **Self-documenting**: Error types reveal failure modes
- **Testability**: Easy to test specific error cases

## Error Design Guidelines

| Guideline | Good | Bad |
|-----------|------|-----|
| **Specific** | `CustomerNotFound(CustomerId)` | `Error("not found")` |
| **Structured** | Error with typed fields | String message |
| **Exhaustive** | Sealed hierarchy | Open-ended exceptions |
| **Domain-focused** | Business error types | Technical exceptions |
| **Actionable** | Error suggests fix | Generic error |

## Why It's a Problem

1. **String parsing**: Brittle error handling with string matching
2. **Lost type information**: Can't pattern match on error types
3. **Missing context**: No structured data about the error
4. **Incomplete handling**: Easy to miss error cases
5. **Poor user experience**: Can't provide specific error messages

## Symptoms

- String comparisons to handle errors
- Error codes as magic numbers or strings
- Catching generic `Exception` types
- Comments explaining error string formats
- Difficulty testing specific error scenarios

## See Also

- [Honest Functions](./honest-functions.md) — Result types for error handling
- [Boolean Blindness](./boolean-blindness.md) — using types instead of booleans
- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md) — discriminated unions
- [Domain Events](./domain-events.md) — events as typed outcomes
- [Smart Constructors](./smart-constructors.md) — validation with Result types
