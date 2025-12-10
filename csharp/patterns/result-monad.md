# Result Monad (Railway-Oriented Programming)

> Using exceptions for control flow or returning null on failure—use Result types to make success and failure explicit in the type system.

## Problem

Traditional error handling uses exceptions (exceptional circumstances only) or returns null/default values (loses error information). Neither approach makes failures explicit in the method signature, leading to forgotten error handling and unclear contracts.

## Example

### ❌ Before

```csharp
// Exceptions: Failure cases hidden in implementation
public User GetUser(int id)
{
    var user = _repository.FindById(id);
    
    if (user == null)
        throw new UserNotFoundException(id);  // Surprise! Not in signature
    
    if (!user.IsActive)
        throw new UserInactiveException(id);  // Another surprise!
    
    return user;
}

// Caller must know to catch specific exceptions
try
{
    var user = GetUser(123);
    ProcessUser(user);
}
catch (UserNotFoundException ex)
{
    // Handle not found
}
catch (UserInactiveException ex)
{
    // Handle inactive
}
catch (Exception ex)
{
    // What other exceptions might be thrown?
}

// Or: Returning null loses error information
public User? GetUser(int id)
{
    var user = _repository.FindById(id);
    
    if (user == null) return null;  // Why? Not found? Error?
    if (!user.IsActive) return null;  // Why? Different reason?
    
    return user;
}

// Caller doesn't know WHY it failed
var user = GetUser(123);
if (user == null)
{
    // Not found? Inactive? Database error? Don't know!
    return BadRequest("User not found... or something");
}
```

**Problems:**
- Exceptions don't appear in signature
- Null return loses error context
- Callers don't know all possible outcomes
- Easy to forget error handling
- Mixing exceptions with happy path logic

### ✅ After (Result Monad)

```csharp
/// <summary>
/// Result of an operation—either success with value or failure with error.
/// </summary>
public abstract record Result<TValue, TError>
{
    private Result() { }
    
    public sealed record Success(TValue Value) : Result<TValue, TError>;
    public sealed record Failure(TError Error) : Result<TValue, TError>;
    
    public bool IsSuccess => this is Success;
    public bool IsFailure => this is Failure;
    
    // Factory methods
    public static Result<TValue, TError> Ok(TValue value) =>
        new Success(value);
    
    public static Result<TValue, TError> Fail(TError error) =>
        new Failure(error);
}

// Extension methods for Result
public static class ResultExtensions
{
    /// <summary>
    /// Match on result—forces handling both cases.
    /// </summary>
    public static TResult Match<TValue, TError, TResult>(
        this Result<TValue, TError> result,
        Func<TValue, TResult> onSuccess,
        Func<TError, TResult> onFailure)
    {
        return result switch
        {
            Result<TValue, TError>.Success success => onSuccess(success.Value),
            Result<TValue, TError>.Failure failure => onFailure(failure.Error),
            _ => throw new InvalidOperationException("Invalid result state")
        };
    }
    
    /// <summary>
    /// Transform success value—failure passes through.
    /// </summary>
    public static Result<TResult, TError> Map<TValue, TError, TResult>(
        this Result<TValue, TError> result,
        Func<TValue, TResult> mapper)
    {
        return result switch
        {
            Result<TValue, TError>.Success success =>
                Result<TResult, TError>.Ok(mapper(success.Value)),
            Result<TValue, TError>.Failure failure =>
                Result<TResult, TError>.Fail(failure.Error),
            _ => throw new InvalidOperationException()
        };
    }
    
    /// <summary>
    /// Chain operations that return Result—stops on first failure.
    /// </summary>
    public static Result<TResult, TError> Bind<TValue, TError, TResult>(
        this Result<TValue, TError> result,
        Func<TValue, Result<TResult, TError>> binder)
    {
        return result switch
        {
            Result<TValue, TError>.Success success =>
                binder(success.Value),
            Result<TValue, TError>.Failure failure =>
                Result<TResult, TError>.Fail(failure.Error),
            _ => throw new InvalidOperationException()
        };
    }
}

// Define specific error types
public abstract record GetUserError
{
    private GetUserError() { }
    
    public sealed record NotFound(UserId UserId) : GetUserError;
    public sealed record Inactive(UserId UserId, string Reason) : GetUserError;
    public sealed record DatabaseError(string Message) : GetUserError;
}

// Method signature reveals all possible outcomes
public Result<User, GetUserError> GetUser(UserId id)
{
    var userResult = _repository.FindById(id);
    
    if (userResult is null)
        return Result<User, GetUserError>.Fail(
            new GetUserError.NotFound(id));
    
    var user = userResult.Value;
    
    if (!user.IsActive)
        return Result<User, GetUserError>.Fail(
            new GetUserError.Inactive(id, user.DeactivationReason));
    
    return Result<User, GetUserError>.Ok(user);
}

// Caller forced to handle both success and failure
var userResult = GetUser(userId);

return userResult.Match(
    onSuccess: user =>
    {
        ProcessUser(user);
        return Ok();
    },
    onFailure: error => error switch
    {
        GetUserError.NotFound notFound =>
            NotFound($"User {notFound.UserId} not found"),
        
        GetUserError.Inactive inactive =>
            BadRequest($"User inactive: {inactive.Reason}"),
        
        GetUserError.DatabaseError dbError =>
            StatusCode(500, dbError.Message)
    });
```

## Chaining Operations (Railway-Oriented Programming)

```csharp
// Each step returns Result—stops on first failure
public Result<OrderConfirmation, ProcessOrderError> ProcessOrder(OrderId orderId)
{
    return GetOrder(orderId)
        .Bind(order => ValidateOrder(order))
        .Bind(validOrder => ReserveInventory(validOrder))
        .Bind(reservation => ProcessPayment(reservation))
        .Bind(payment => ShipOrder(payment))
        .Map(shipment => new OrderConfirmation(shipment.TrackingNumber));
}

Result<Order, ProcessOrderError> GetOrder(OrderId id)
{
    var orderOption = _repository.GetById(id);
    
    return orderOption.HasValue
        ? Result<Order, ProcessOrderError>.Ok(orderOption.Value)
        : Result<Order, ProcessOrderError>.Fail(
            new ProcessOrderError.OrderNotFound(id));
}

Result<ValidatedOrder, ProcessOrderError> ValidateOrder(Order order)
{
    if (order.Items.Count == 0)
        return Result<ValidatedOrder, ProcessOrderError>.Fail(
            new ProcessOrderError.OrderEmpty(order.Id));
    
    if (order.Total < Money.USD(0.01m))
        return Result<ValidatedOrder, ProcessOrderError>.Fail(
            new ProcessOrderError.InvalidTotal(order.Id, order.Total));
    
    return Result<ValidatedOrder, ProcessOrderError>.Ok(
        new ValidatedOrder(order));
}

Result<InventoryReservation, ProcessOrderError> ReserveInventory(ValidatedOrder order)
{
    foreach (var item in order.Items)
    {
        var available = _inventory.GetAvailable(item.ProductId);
        
        if (available < item.Quantity)
            return Result<InventoryReservation, ProcessOrderError>.Fail(
                new ProcessOrderError.InsufficientInventory(
                    order.Id,
                    item.ProductId,
                    available,
                    item.Quantity));
    }
    
    var reservationId = _inventory.Reserve(order.Items);
    return Result<InventoryReservation, ProcessOrderError>.Ok(
        new InventoryReservation(order, reservationId));
}

// If any step fails, subsequent steps are skipped
// Error information flows through the chain
```

## Async Result Operations

```csharp
public static class AsyncResultExtensions
{
    public static async Task<Result<TResult, TError>> MapAsync<TValue, TError, TResult>(
        this Task<Result<TValue, TError>> resultTask,
        Func<TValue, TResult> mapper)
    {
        var result = await resultTask;
        return result.Map(mapper);
    }
    
    public static async Task<Result<TResult, TError>> BindAsync<TValue, TError, TResult>(
        this Task<Result<TValue, TError>> resultTask,
        Func<TValue, Task<Result<TResult, TError>>> binder)
    {
        var result = await resultTask;
        
        return result switch
        {
            Result<TValue, TError>.Success success =>
                await binder(success.Value),
            Result<TValue, TError>.Failure failure =>
                Result<TResult, TError>.Fail(failure.Error),
            _ => throw new InvalidOperationException()
        };
    }
    
    public static async Task<TResult> MatchAsync<TValue, TError, TResult>(
        this Task<Result<TValue, TError>> resultTask,
        Func<TValue, Task<TResult>> onSuccess,
        Func<TError, Task<TResult>> onFailure)
    {
        var result = await resultTask;
        
        return result switch
        {
            Result<TValue, TError>.Success success =>
                await onSuccess(success.Value),
            Result<TValue, TError>.Failure failure =>
                await onFailure(failure.Error),
            _ => throw new InvalidOperationException()
        };
    }
}

// Async chain
public async Task<Result<OrderConfirmation, ProcessOrderError>> ProcessOrderAsync(OrderId orderId)
{
    return await GetOrderAsync(orderId)
        .BindAsync(order => ValidateOrderAsync(order))
        .BindAsync(validOrder => ReserveInventoryAsync(validOrder))
        .BindAsync(reservation => ProcessPaymentAsync(reservation))
        .BindAsync(payment => ShipOrderAsync(payment))
        .MapAsync(shipment => new OrderConfirmation(shipment.TrackingNumber));
}
```

## Combining Multiple Results

```csharp
/// <summary>
/// Combine multiple results—success only if all succeed.
/// </summary>
public static Result<IReadOnlyList<TValue>, TError> Sequence<TValue, TError>(
    this IEnumerable<Result<TValue, TError>> results)
{
    var values = new List<TValue>();
    
    foreach (var result in results)
    {
        if (result is Result<TValue, TError>.Failure failure)
            return Result<IReadOnlyList<TValue>, TError>.Fail(failure.Error);
        
        if (result is Result<TValue, TError>.Success success)
            values.Add(success.Value);
    }
    
    return Result<IReadOnlyList<TValue>, TError>.Ok(values);
}

/// <summary>
/// Collect all errors from multiple results.
/// </summary>
public static Result<IReadOnlyList<TValue>, IReadOnlyList<TError>> SequenceAll<TValue, TError>(
    this IEnumerable<Result<TValue, TError>> results)
{
    var values = new List<TValue>();
    var errors = new List<TError>();
    
    foreach (var result in results)
    {
        switch (result)
        {
            case Result<TValue, TError>.Success success:
                values.Add(success.Value);
                break;
            
            case Result<TValue, TError>.Failure failure:
                errors.Add(failure.Error);
                break;
        }
    }
    
    return errors.Count == 0
        ? Result<IReadOnlyList<TValue>, IReadOnlyList<TError>>.Ok(values)
        : Result<IReadOnlyList<TValue>, IReadOnlyList<TError>>.Fail(errors);
}

// Usage: Validate multiple items
public Result<ValidatedOrder, ValidationError> ValidateOrderItems(Order order)
{
    var itemResults = order.Items
        .Select(item => ValidateItem(item))
        .ToList();
    
    return itemResults.Sequence()
        .Map(validItems => new ValidatedOrder(order.Id, validItems));
}
```

## Result with Multiple Error Types

```csharp
/// <summary>
/// Combine results with different error types.
/// </summary>
public static Result<TValue, TErrorB> MapError<TValue, TErrorA, TErrorB>(
    this Result<TValue, TErrorA> result,
    Func<TErrorA, TErrorB> errorMapper)
{
    return result switch
    {
        Result<TValue, TErrorA>.Success success =>
            Result<TValue, TErrorB>.Ok(success.Value),
        Result<TValue, TErrorA>.Failure failure =>
            Result<TValue, TErrorB>.Fail(errorMapper(failure.Error)),
        _ => throw new InvalidOperationException()
    };
}

// Define common error type
public abstract record ApplicationError
{
    private ApplicationError() { }
    
    public sealed record Validation(string Message) : ApplicationError;
    public sealed record NotFound(string Resource, string Id) : ApplicationError;
    public sealed record Unauthorized(string Reason) : ApplicationError;
    public sealed record External(string Service, string Message) : ApplicationError;
}

// Convert specific error to common error
public Result<Order, ApplicationError> GetOrderForProcessing(OrderId id)
{
    return GetOrder(id)
        .MapError<Order, GetOrderError, ApplicationError>(error => error switch
        {
            GetOrderError.NotFound notFound =>
                new ApplicationError.NotFound("Order", notFound.OrderId.ToString()),
            
            GetOrderError.DatabaseError dbError =>
                new ApplicationError.External("Database", dbError.Message)
        })
        .Bind(order => ValidateOrder(order)
            .MapError<ValidatedOrder, ValidationError, ApplicationError>(valError =>
                new ApplicationError.Validation(valError.Message)));
}
```

## Try Pattern for Exceptions

```csharp
/// <summary>
/// Convert exception-throwing code to Result.
/// </summary>
public static Result<TValue, Exception> Try<TValue>(Func<TValue> func)
{
    try
    {
        return Result<TValue, Exception>.Ok(func());
    }
    catch (Exception ex)
    {
        return Result<TValue, Exception>.Fail(ex);
    }
}

/// <summary>
/// Async Try.
/// </summary>
public static async Task<Result<TValue, Exception>> TryAsync<TValue>(
    Func<Task<TValue>> func)
{
    try
    {
        var value = await func();
        return Result<TValue, Exception>.Ok(value);
    }
    catch (Exception ex)
    {
        return Result<TValue, Exception>.Fail(ex);
    }
}

// Usage: Wrap exception-throwing code
var parseResult = Result<int, Exception>.Try(() => int.Parse(input));

var httpResult = await Result<string, Exception>.TryAsync(async () =>
{
    var response = await _httpClient.GetAsync(url);
    response.EnsureSuccessStatusCode();
    return await response.Content.ReadAsStringAsync();
});
```

## Converting Result to HTTP Response

```csharp
public static class ResultHttpExtensions
{
    public static IActionResult ToActionResult<TValue, TError>(
        this Result<TValue, TError> result,
        Func<TValue, IActionResult> onSuccess,
        Func<TError, IActionResult> onFailure)
    {
        return result.Match(onSuccess, onFailure);
    }
    
    public static IActionResult ToActionResult<TValue>(
        this Result<TValue, ApplicationError> result)
    {
        return result.Match(
            onSuccess: value => new OkObjectResult(value),
            onFailure: error => error switch
            {
                ApplicationError.Validation validation =>
                    new BadRequestObjectResult(new { Message = validation.Message }),
                
                ApplicationError.NotFound notFound =>
                    new NotFoundObjectResult(new
                    {
                        Message = $"{notFound.Resource} {notFound.Id} not found"
                    }),
                
                ApplicationError.Unauthorized unauthorized =>
                    new ObjectResult(new { Message = unauthorized.Reason })
                    {
                        StatusCode = 403
                    },
                
                ApplicationError.External external =>
                    new ObjectResult(new
                    {
                        Message = $"External service error: {external.Service}",
                        Details = external.Message
                    })
                    {
                        StatusCode = 502
                    }
            });
    }
}

// Usage in controller
public IActionResult GetOrder(Guid orderId)
{
    var result = _orderService.GetOrder(new OrderId(orderId));
    
    return result.ToActionResult();
}
```

## Key Principles

### 1. Make Failure Explicit

```csharp
// ❌ Hidden failure
User GetUser(int id);

// ✅ Explicit failure
Result<User, GetUserError> GetUser(UserId id);
```

### 2. Chain Operations

```csharp
// Stop on first failure, pass success through
return GetOrder(id)
    .Bind(Validate)
    .Bind(Process)
    .Map(CreateConfirmation);
```

### 3. Exhaustive Error Handling

```csharp
// Compiler forces handling all error cases
result.Match(
    onSuccess: value => { /* handle success */ },
    onFailure: error => error switch
    {
        Error.A => { /* handle A */ },
        Error.B => { /* handle B */ }
        // Compiler error if missing case
    });
```

## Why It's a Problem

1. **Hidden failures**: Exceptions don't appear in signature
2. **Lost context**: Null return doesn't explain why
3. **Forgotten handling**: Easy to forget try-catch
4. **Unclear contracts**: Caller doesn't know possible outcomes
5. **Performance**: Exceptions are expensive

## Symptoms

- Try-catch blocks throughout code
- Null checks everywhere
- Comments like "throws XException when..."
- Methods returning null for errors
- Inconsistent error handling

## Benefits

- **Explicit contracts**: Signature shows all outcomes
- **Forced handling**: Cannot ignore errors
- **Composable**: Chain operations cleanly
- **Type-safe**: Compiler ensures exhaustive handling
- **Performance**: No exception overhead

## Trade-offs

- **Verbosity**: More code than exceptions
- **Learning curve**: Monadic operations unfamiliar
- **Library integration**: Some libraries expect exceptions
- **Nesting**: Deeply nested Result types can be complex

## See Also

- [Honest Functions](./honest-functions.md) — explicit outcomes in signatures
- [Typed Errors](./typed-errors.md) — error cases as types
- [Discriminated Unions](./discriminated-unions.md) — modeling alternatives
- [Exhaustive Pattern Matching](./exhaustive-pattern-matching.md) — compiler-enforced completeness
