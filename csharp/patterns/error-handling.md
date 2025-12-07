# Error Handling

> Use exceptions for exceptional circumstances; use Result types for expected failures.

## Problem

Mixing exceptions with business logic makes code unpredictable. Using exceptions for flow control is expensive and hides the expected error paths. Ignoring errors or swallowing exceptions leads to silent failures.

## Example

### ❌ Before

```csharp
public class OrderService
{
    public void ProcessOrder(int orderId)
    {
        try
        {
            var order = _repository.GetOrder(orderId);
            
            if (order == null)
                throw new OrderNotFoundException(orderId);

            if (order.Total <= 0)
                throw new InvalidOrderException("Order total must be positive");

            if (!HasInventory(order))
                throw new InsufficientInventoryException();

            var payment = _paymentGateway.Charge(order.Total);
            
            if (!payment.Success)
                throw new PaymentFailedException(payment.Error);

            order.Status = OrderStatus.Completed;
            _repository.Update(order);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing order");
            throw;  // Rethrow without adding context
        }
    }
}

// Caller has to guess what exceptions might be thrown
try
{
    orderService.ProcessOrder(orderId);
}
catch (OrderNotFoundException) { }
catch (InvalidOrderException) { }
catch (InsufficientInventoryException) { }
catch (PaymentFailedException) { }
catch (Exception) { }  // Catch-all for unexpected errors
```

**Problems:**
- Expected failures (user error, business rules) use exceptions
- Expensive stack trace generation for common scenarios
- Caller doesn't know which exceptions to handle
- Generic catch blocks hide specific errors
- Hard to compose error handling logic

### ✅ After

```csharp
// Define error types for expected failures
public abstract record ProcessOrderError;
public sealed record OrderNotFound(OrderId Id) : ProcessOrderError;
public sealed record InvalidOrderTotal(OrderId Id, Money Total) : ProcessOrderError;
public sealed record InsufficientInventory(OrderId Id, ProductId ProductId) : ProcessOrderError;
public sealed record PaymentFailed(OrderId Id, string Reason) : ProcessOrderError;

public class OrderService
{
    public Result<Order, ProcessOrderError> ProcessOrder(OrderId orderId)
    {
        return FindOrder(orderId)
            .Bind(ValidateOrder)
            .Bind(CheckInventory)
            .Bind(ProcessPayment)
            .Bind(CompleteOrder);
    }

    private Result<Order, ProcessOrderError> FindOrder(OrderId id)
    {
        var order = _repository.FindById(id);
        return order != null
            ? Result.Success<Order, ProcessOrderError>(order)
            : Result.Failure<Order, ProcessOrderError>(new OrderNotFound(id));
    }

    private Result<Order, ProcessOrderError> ValidateOrder(Order order)
    {
        return order.Total.Amount > 0
            ? Result.Success<Order, ProcessOrderError>(order)
            : Result.Failure<Order, ProcessOrderError>(new InvalidOrderTotal(order.Id, order.Total));
    }

    // ... other methods
}

// Caller explicitly handles all error cases
var result = orderService.ProcessOrder(orderId);

result.Match(
    onSuccess: order => Console.WriteLine($"Order {order.Id} completed"),
    onFailure: error => error switch
    {
        OrderNotFound e => LogWarning($"Order {e.Id} not found"),
        InvalidOrderTotal e => LogWarning($"Invalid total: {e.Total}"),
        InsufficientInventory e => LogWarning($"Out of stock: {e.ProductId}"),
        PaymentFailed e => LogError($"Payment failed: {e.Reason}"),
        _ => LogError("Unknown error")
    }
);
```

## When to Use Exceptions

### ✅ Use Exceptions For

**1. Programming Errors**
```csharp
// Violates contract—should never happen in correct code
public Money Divide(Money amount, decimal divisor)
{
    if (divisor == 0)
        throw new ArgumentException("Divisor cannot be zero", nameof(divisor));

    return amount.Divide(divisor);
}
```

**2. System/Infrastructure Failures**
```csharp
// Database down, network timeout, out of memory
public Order GetOrder(OrderId id)
{
    try
    {
        return _database.Query<Order>(id);
    }
    catch (SqlException ex)
    {
        // Infrastructure failure—let it bubble up
        throw new DatabaseException("Failed to retrieve order", ex);
    }
}
```

**3. Third-Party Library Exceptions**
```csharp
// Can't control how libraries report errors
public void ParseJson(string json)
{
    try
    {
        return JsonSerializer.Deserialize<Order>(json);
    }
    catch (JsonException ex)
    {
        throw new InvalidDataException("Invalid JSON format", ex);
    }
}
```

### ❌ Don't Use Exceptions For

**1. Expected Business Failures**
```csharp
// ❌ User not found is expected
public User GetUser(UserId id)
{
    var user = _repository.Find(id);
    if (user == null)
        throw new UserNotFoundException(id);
    return user;
}

// ✅ Use Result type
public Result<User, UserError> GetUser(UserId id)
{
    var user = _repository.Find(id);
    return user != null
        ? Result.Success<User, UserError>(user)
        : Result.Failure<User, UserError>(new UserNotFound(id));
}
```

**2. Validation Failures**
```csharp
// ❌ Invalid input is expected
public void UpdateProfile(string email)
{
    if (!IsValidEmail(email))
        throw new ValidationException("Invalid email");
    // ...
}

// ✅ Use Result type
public Result<Unit, ValidationError> UpdateProfile(string email)
{
    return Email.Create(email)
        .Bind(validEmail => _repository.UpdateEmail(userId, validEmail));
}
```

**3. Flow Control**
```csharp
// ❌ Exception for control flow
try
{
    var value = int.Parse(input);
    ProcessValue(value);
}
catch (FormatException)
{
    Console.WriteLine("Invalid number");
}

// ✅ Use TryParse
if (int.TryParse(input, out var value))
{
    ProcessValue(value);
}
else
{
    Console.WriteLine("Invalid number");
}
```

## Exception Handling Best Practices

### Catch Specific Exceptions

```csharp
// ❌ Generic catch hides different failure modes
try
{
    ProcessFile(path);
}
catch (Exception ex)
{
    _logger.LogError(ex, "File processing failed");
}

// ✅ Handle specific exceptions differently
try
{
    ProcessFile(path);
}
catch (FileNotFoundException ex)
{
    _logger.LogWarning(ex, "File not found: {Path}", path);
    return Result.Failure(new FileNotFound(path));
}
catch (UnauthorizedAccessException ex)
{
    _logger.LogError(ex, "Access denied: {Path}", path);
    return Result.Failure(new AccessDenied(path));
}
catch (IOException ex)
{
    _logger.LogError(ex, "I/O error reading file: {Path}", path);
    throw;  // Infrastructure error—let it bubble
}
```

### Preserve Stack Traces

```csharp
// ❌ Loses original stack trace
try
{
    ProcessOrder(order);
}
catch (Exception ex)
{
    _logger.LogError(ex.Message);
    throw ex;  // Bad: resets stack trace
}

// ✅ Preserves stack trace
try
{
    ProcessOrder(order);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Failed to process order");
    throw;  // Good: preserves original stack trace
}

// ✅ Add context with new exception
try
{
    ProcessOrder(order);
}
catch (Exception ex)
{
    throw new OrderProcessingException($"Failed to process order {order.Id}", ex);
}
```

### Don't Swallow Exceptions

```csharp
// ❌ Silent failure
try
{
    SendEmail(email);
}
catch (Exception)
{
    // Swallowing exceptions hides problems
}

// ✅ Log and handle appropriately
try
{
    SendEmail(email);
}
catch (SmtpException ex)
{
    _logger.LogError(ex, "Failed to send email to {Recipient}", email.Recipient);
    _emailQueue.Retry(email);  // Handle the failure
}
```

### Finally Blocks for Cleanup

```csharp
// ❌ Resource leak if exception occurs
var file = File.Open(path, FileMode.Open);
ProcessFile(file);
file.Close();

// ✅ Using finally
FileStream file = null;
try
{
    file = File.Open(path, FileMode.Open);
    ProcessFile(file);
}
finally
{
    file?.Close();
}

// ✅ Even better: using statement
using var file = File.Open(path, FileMode.Open);
ProcessFile(file);
```

## Result Type Pattern

### Simple Result

```csharp
public readonly record struct Result<T>
{
    private readonly T? value;
    private readonly Exception? error;

    public bool IsSuccess { get; }
    public T Value => IsSuccess ? value! : throw new InvalidOperationException("Result is failure");
    public Exception Error => !IsSuccess ? error! : throw new InvalidOperationException("Result is success");

    private Result(T value)
    {
        this.value = value;
        this.error = null;
        IsSuccess = true;
    }

    private Result(Exception error)
    {
        this.value = default;
        this.error = error;
        IsSuccess = false;
    }

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(Exception error) => new(error);

    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<Exception, TResult> onFailure)
        => IsSuccess ? onSuccess(value!) : onFailure(error!);
}
```

### Typed Errors

```csharp
public readonly record struct Result<TValue, TError>
{
    private readonly TValue? value;
    private readonly TError? error;

    public bool IsSuccess { get; }

    private Result(TValue value)
    {
        this.value = value;
        this.error = default;
        IsSuccess = true;
    }

    private Result(TError error)
    {
        this.value = default;
        this.error = error;
        IsSuccess = false;
    }

    public static Result<TValue, TError> Success(TValue value) => new(value);
    public static Result<TValue, TError> Failure(TError error) => new(error);

    public Result<TResult, TError> Map<TResult>(Func<TValue, TResult> mapper)
    {
        return IsSuccess
            ? Result<TResult, TError>.Success(mapper(value!))
            : Result<TResult, TError>.Failure(error!);
    }

    public Result<TResult, TError> Bind<TResult>(Func<TValue, Result<TResult, TError>> binder)
    {
        return IsSuccess
            ? binder(value!)
            : Result<TResult, TError>.Failure(error!);
    }

    public TResult Match<TResult>(Func<TValue, TResult> onSuccess, Func<TError, TResult> onFailure)
        => IsSuccess ? onSuccess(value!) : onFailure(error!);
}
```

## Global Exception Handling

### ASP.NET Core Middleware

```csharp
public class GlobalExceptionHandler : IMiddleware
{
    private readonly ILogger<GlobalExceptionHandler> logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        this.logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        try
        {
            await next(context);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Unhandled exception occurred");
            
            var problemDetails = ex switch
            {
                ValidationException => CreateProblemDetails(400, "Validation failed", ex.Message),
                NotFoundException => CreateProblemDetails(404, "Not found", ex.Message),
                UnauthorizedAccessException => CreateProblemDetails(403, "Forbidden", ex.Message),
                _ => CreateProblemDetails(500, "Internal server error", "An error occurred")
            };

            context.Response.StatusCode = problemDetails.Status ?? 500;
            await context.Response.WriteAsJsonAsync(problemDetails);
        }
    }

    private ProblemDetails CreateProblemDetails(int status, string title, string detail)
    {
        return new ProblemDetails
        {
            Status = status,
            Title = title,
            Detail = detail
        };
    }
}
```

## Symptoms

- Try-catch blocks everywhere
- Empty catch blocks
- Generic `catch (Exception)` handlers
- Exceptions used for business logic
- Methods that throw undocumented exceptions
- Silent failures (swallowed exceptions)

## Benefits

- **Explicit error handling**: Compiler enforces handling of expected failures
- **Better performance**: No stack trace generation for expected failures
- **Type-safe errors**: Exhaustive pattern matching on error types
- **Self-documenting**: Method signatures reveal possible failures
- **Composable**: Result types chain naturally with `Map` and `Bind`

## See Also

- [Honest Functions](./honest-functions.md) — Making error cases explicit in signatures
- [Nullability vs. Optionality](./nullability-optionality.md) — Using Option types for missing values
- [Primitive Obsession](./primitive-obsession.md) — Using typed errors instead of strings
