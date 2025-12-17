# Exception Handling Best Practices

> Use exceptions for exceptional circumstances—not for control flow. Catch specific exceptions and handle them appropriately.

## Problem

Catching broad exceptions, using exceptions for control flow, or silently swallowing errors leads to hard-to-debug applications and masks real problems.

## Example

### ❌ Before

```csharp
public User GetUser(string userId)
{
    try
    {
        var user = _database.Query($"SELECT * FROM Users WHERE Id = '{userId}'");
        return user;
    }
    catch (Exception ex)
    {
        // Swallow all exceptions
        return null;
    }
}

public bool IsValidEmail(string email)
{
    try
    {
        var addr = new MailAddress(email);
        return true;
    }
    catch
    {
        return false;  // Using exceptions for control flow
    }
}
```

### ✅ After

```csharp
public Result<User, GetUserError> GetUser(UserId userId)
{
    try
    {
        var user = _database.Query(userId);
        return user != null
            ? Result.Success<User, GetUserError>(user)
            : Result.Failure<User, GetUserError>(new UserNotFound(userId));
    }
    catch (DatabaseConnectionException ex)
    {
        _logger.LogError(ex, "Database connection failed for user {UserId}", userId);
        return Result.Failure<User, GetUserError>(new DatabaseUnavailable());
    }
    catch (DatabaseTimeoutException ex)
    {
        _logger.LogWarning(ex, "Database timeout for user {UserId}", userId);
        return Result.Failure<User, GetUserError>(new DatabaseTimeout());
    }
}

public bool IsValidEmail(EmailAddress email)
{
    return EmailAddress.TryCreate(email.Value, out _);
}

// Or use a proper validation approach
public sealed record EmailAddress
{
    public string Value { get; }

    private EmailAddress(string value)
    {
        this.Value = value;
    }

    public static Result<EmailAddress, ValidationError> Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
        {
            return Result.Failure<EmailAddress, ValidationError>(
                new ValidationError("Email cannot be empty"));
        }

        if (!IsValidFormat(value))
        {
            return Result.Failure<EmailAddress, ValidationError>(
                new ValidationError("Invalid email format"));
        }

        return Result.Success<EmailAddress, ValidationError>(new EmailAddress(value));
    }

    private static bool IsValidFormat(string email)
    {
        // Use regex or other validation logic, not exceptions
        return email.Contains("@") && email.Contains(".");
    }
}
```

## Best Practices

### Catch Specific Exceptions

```csharp
// ❌ Too broad
try
{
    ProcessPayment(order);
}
catch (Exception ex)
{
    LogError(ex);
}

// ✅ Specific and actionable
try
{
    ProcessPayment(order);
}
catch (PaymentGatewayException ex)
{
    _logger.LogError(ex, "Payment gateway failed for order {OrderId}", order.Id);
    return Result.Failure<Unit>(new PaymentFailed(ex.Message));
}
catch (InsufficientFundsException ex)
{
    _logger.LogWarning("Insufficient funds for order {OrderId}", order.Id);
    return Result.Failure<Unit>(new InsufficientFunds());
}
```

### Never Swallow Exceptions

```csharp
// ❌ Silent failure
try
{
    SaveToDatabase(data);
}
catch
{
    // Silent failure - data loss!
}

// ✅ Log and propagate or handle appropriately
try
{
    SaveToDatabase(data);
}
catch (DatabaseException ex)
{
    _logger.LogError(ex, "Failed to save data");
    throw new DataPersistenceException("Could not save data", ex);
}
```

### Don't Use Exceptions for Control Flow

```csharp
// ❌ Exception-driven logic
public int ParseAge(string input)
{
    try
    {
        return int.Parse(input);
    }
    catch
    {
        return 0;
    }
}

// ✅ Use TryParse pattern
public Option<Age> ParseAge(string input)
{
    return int.TryParse(input, out var value) && value >= 0
        ? Option.Some(Age.Create(value))
        : Option.None<Age>();
}
```

### Preserve Stack Traces

```csharp
// ❌ Loses stack trace
try
{
    ProcessData();
}
catch (Exception ex)
{
    throw ex;  // Resets stack trace!
}

// ✅ Preserves stack trace
try
{
    ProcessData();
}
catch (Exception ex)
{
    throw;  // Preserves original stack trace
}

// ✅ Wrap with context
try
{
    ProcessData();
}
catch (Exception ex)
{
    throw new DataProcessingException("Failed to process data", ex);
}
```

### Use Finally for Cleanup (or better, use using)

```csharp
// ❌ Resource leak risk
FileStream stream = null;
try
{
    stream = File.OpenRead(path);
    ProcessFile(stream);
}
catch (IOException ex)
{
    LogError(ex);
}
finally
{
    stream?.Dispose();
}

// ✅ Using statement guarantees cleanup
try
{
    using var stream = File.OpenRead(path);
    ProcessFile(stream);
}
catch (IOException ex)
{
    LogError(ex);
    throw new FileProcessingException("Failed to process file", ex);
}
```

### Provide Context in Exception Messages

```csharp
// ❌ Vague error message
throw new InvalidOperationException("Invalid state");

// ✅ Helpful context
throw new InvalidOperationException(
    $"Cannot process order {orderId} in {order.Status} status. " +
    $"Expected status: {OrderStatus.Pending}");
```

### Create Custom Exception Types for Domain Errors

```csharp
// ✅ Domain-specific exceptions
public sealed class OrderNotFoundException : Exception
{
    public OrderId OrderId { get; }

    public OrderNotFoundException(OrderId orderId)
        : base($"Order {orderId} was not found")
    {
        this.OrderId = orderId;
    }
}

public sealed class InsufficientInventoryException : Exception
{
    public ProductId ProductId { get; }
    public int Available { get; }
    public int Requested { get; }

    public InsufficientInventoryException(
        ProductId productId, 
        int available, 
        int requested)
        : base($"Insufficient inventory for product {productId}. " +
               $"Available: {available}, Requested: {requested}")
    {
        this.ProductId = productId;
        this.Available = available;
        this.Requested = requested;
    }
}
```

### Prefer Result Types Over Exceptions

```csharp
// ❌ Exception-heavy
public User GetUserById(UserId userId)
{
    var user = _repository.Find(userId);
    if (user == null)
    {
        throw new UserNotFoundException(userId);
    }
    return user;
}

// ✅ Result type for expected failures
public Result<User, UserError> GetUserById(UserId userId)
{
    var user = _repository.Find(userId);
    return user != null
        ? Result.Success<User, UserError>(user)
        : Result.Failure<User, UserError>(new UserNotFound(userId));
}
```

## Symptoms

- Try-catch blocks with empty catch handlers
- Catching `Exception` instead of specific exception types
- Using exceptions to validate input
- Lost exception context and stack traces
- Resource leaks from improper cleanup
- Generic error messages that don't help debugging

## Benefits

- **Easier debugging** with preserved stack traces and context
- **Clearer error handling** by catching specific exceptions
- **Better performance** by avoiding exception-based control flow
- **Safer code** that doesn't silently swallow errors
- **Domain clarity** through custom exception types

## See Also

- [Honest Functions](./honest-functions.md) — Using Result types instead of exceptions
- [Typed Errors](./typed-errors.md) — Making failure cases explicit in types
- [Result Monad](./result-monad.md) — Composable error handling
