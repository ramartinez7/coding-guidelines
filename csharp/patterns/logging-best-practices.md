# Logging Best Practices

> Structure logs for searchability, use appropriate log levels, avoid logging sensitive data, and leverage structured logging.

## Problem

Poor logging practices lead to unsearchable logs, performance degradation, security vulnerabilities, and difficulty troubleshooting production issues.

## Example

### ❌ Before

```csharp
public class OrderService
{
    public void ProcessOrder(Order order)
    {
        Console.WriteLine("Processing order");  // Wrong output method
        
        var total = order.Items.Sum(i => i.Price);
        Console.WriteLine("Total: " + total);  // String concatenation
        
        _logger.LogInformation($"Order {order.Id} processed by {order.Customer.Email}");  // Sensitive data
        
        try
        {
            SaveOrder(order);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex.Message);  // Loses stack trace
        }
    }
}
```

### ✅ After

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public Result<Unit, ProcessOrderError> ProcessOrder(Order order)
    {
        _logger.LogInformation(
            "Processing order {OrderId} for customer {CustomerId}",
            order.Id,
            order.CustomerId);  // Structured logging, no sensitive data

        var total = order.Items.Sum(i => i.Price);

        _logger.LogDebug(
            "Calculated total {Total} for order {OrderId}",
            total,
            order.Id);

        try
        {
            SaveOrder(order);

            _logger.LogInformation(
                "Successfully processed order {OrderId}",
                order.Id);

            return Result.Success<Unit, ProcessOrderError>(Unit.Value);
        }
        catch (DatabaseException ex)
        {
            _logger.LogError(
                ex,
                "Failed to save order {OrderId}",
                order.Id);  // Preserves exception

            return Result.Failure<Unit, ProcessOrderError>(
                new DatabaseError(ex.Message));
        }
    }
}
```

## Best Practices

### 1. Use Structured Logging

```csharp
// ❌ String interpolation (not searchable)
_logger.LogInformation($"User {userId} logged in at {DateTime.Now}");

// ✅ Structured logging (searchable by field)
_logger.LogInformation(
    "User {UserId} logged in at {LoginTime}",
    userId,
    DateTime.UtcNow);
```

### 2. Use Appropriate Log Levels

```csharp
// ❌ Everything as Information
_logger.LogInformation("Starting process");
_logger.LogInformation("Error occurred: database timeout");
_logger.LogInformation("Debug: variable x = 5");

// ✅ Correct log levels
_logger.LogDebug("Variable x = {X}", 5);  // Development details
_logger.LogInformation("Processing order {OrderId}", orderId);  // Important events
_logger.LogWarning("Retry attempt {Attempt} for order {OrderId}", attempt, orderId);
_logger.LogError(ex, "Failed to process payment for order {OrderId}", orderId);
_logger.LogCritical(ex, "Database connection pool exhausted");  // System failure
```

### 3. Don't Log Sensitive Data

```csharp
// ❌ Logging sensitive information
_logger.LogInformation("User logged in with password {Password}", password);
_logger.LogInformation("Processing payment with card {CardNumber}", cardNumber);
_logger.LogInformation("API key: {ApiKey}", apiKey);

// ✅ Log only safe identifiers
_logger.LogInformation("User {UserId} logged in", userId);
_logger.LogInformation(
    "Processing payment with card ending {Last4}",
    cardNumber.Last4Digits);
_logger.LogInformation("API key validated for client {ClientId}", clientId);
```

### 4. Include Exception Objects

```csharp
// ❌ Only message (loses stack trace)
catch (Exception ex)
{
    _logger.LogError(ex.Message);
}

// ❌ String formatting
catch (Exception ex)
{
    _logger.LogError($"Error: {ex}");  // Not structured
}

// ✅ Pass exception as first parameter
catch (Exception ex)
{
    _logger.LogError(
        ex,
        "Failed to process order {OrderId}",
        orderId);
}
```

### 5. Add Correlation IDs

```csharp
// ❌ No correlation between related logs
public async Task ProcessOrder(OrderId orderId)
{
    _logger.LogInformation("Processing order {OrderId}", orderId);
    await ValidateInventory(orderId);
    await ProcessPayment(orderId);
}

// ✅ Include correlation ID for distributed tracing
public async Task ProcessOrder(OrderId orderId, CorrelationId correlationId)
{
    using (_logger.BeginScope(new Dictionary<string, object>
    {
        ["CorrelationId"] = correlationId,
        ["OrderId"] = orderId
    }))
    {
        _logger.LogInformation("Processing order");
        await ValidateInventory(orderId);
        await ProcessPayment(orderId);
    }
}
```

### 6. Use Log Scopes for Context

```csharp
// ❌ Repeating context in every log
_logger.LogInformation("Processing order {OrderId} step 1", orderId);
_logger.LogInformation("Processing order {OrderId} step 2", orderId);
_logger.LogInformation("Processing order {OrderId} step 3", orderId);

// ✅ Use scope to add context
using (_logger.BeginScope("Processing order {OrderId}", orderId))
{
    _logger.LogInformation("Validating inventory");
    _logger.LogInformation("Processing payment");
    _logger.LogInformation("Confirming order");
}
```

### 7. Guard Expensive Logging

```csharp
// ❌ Always evaluates ToString() even if debug disabled
_logger.LogDebug("Complex object: {Object}", complexObject.ToString());

// ✅ Check if level enabled before expensive operations
if (_logger.IsEnabled(LogLevel.Debug))
{
    var serialized = JsonSerializer.Serialize(complexObject);
    _logger.LogDebug("Complex object: {Object}", serialized);
}

// ✅ Or use log level compile-time constants
_logger.LogDebug("Complex object: {@Object}", complexObject);  // Serilog
```

### 8. Don't Log in Hot Paths

```csharp
// ❌ Logging in tight loop
public void ProcessItems(List<Item> items)
{
    foreach (var item in items)
    {
        _logger.LogDebug("Processing item {ItemId}", item.Id);  // Too chatty!
        ProcessItem(item);
    }
}

// ✅ Log summary instead
public void ProcessItems(List<Item> items)
{
    _logger.LogInformation("Processing {ItemCount} items", items.Count);

    foreach (var item in items)
    {
        ProcessItem(item);
    }

    _logger.LogInformation("Completed processing {ItemCount} items", items.Count);
}
```

### 9. Use Semantic Logging

```csharp
// ❌ Generic messages
_logger.LogInformation("Operation failed");

// ✅ Specific, actionable messages
_logger.LogWarning(
    "Payment gateway timeout after {Timeout}ms for order {OrderId}. " +
    "Will retry {RemainingRetries} more times",
    timeoutMs,
    orderId,
    remainingRetries);
```

### 10. Create Logging Extensions

```csharp
// ✅ Create extension methods for common log patterns
public static class LoggerExtensions
{
    public static void LogOrderProcessed(
        this ILogger logger,
        OrderId orderId,
        decimal total,
        TimeSpan duration)
    {
        logger.LogInformation(
            "Order {OrderId} processed successfully. " +
            "Total: {Total:C}, Duration: {Duration}ms",
            orderId,
            total,
            duration.TotalMilliseconds);
    }

    public static void LogPaymentFailed(
        this ILogger logger,
        OrderId orderId,
        string reason,
        Exception? exception = null)
    {
        logger.LogError(
            exception,
            "Payment failed for order {OrderId}. Reason: {Reason}",
            orderId,
            reason);
    }
}

// Usage
_logger.LogOrderProcessed(orderId, total, duration);
_logger.LogPaymentFailed(orderId, "Insufficient funds", ex);
```

### 11. Log Performance Metrics

```csharp
// ✅ Track performance of operations
public async Task<Order> ProcessOrder(OrderId orderId)
{
    var sw = Stopwatch.StartNew();

    try
    {
        var order = await _repository.GetOrderAsync(orderId);
        await ProcessPaymentAsync(order);

        sw.Stop();

        _logger.LogInformation(
            "Order {OrderId} processed in {ElapsedMs}ms",
            orderId,
            sw.ElapsedMilliseconds);

        return order;
    }
    catch (Exception ex)
    {
        sw.Stop();

        _logger.LogError(
            ex,
            "Order {OrderId} failed after {ElapsedMs}ms",
            orderId,
            sw.ElapsedMilliseconds);

        throw;
    }
}
```

### 12. Use LoggerMessage for High-Performance

```csharp
// ❌ Allocations on every call
_logger.LogInformation("Processing order {OrderId}", orderId);

// ✅ Compile-time generated code (zero allocations)
public partial class OrderService
{
    [LoggerMessage(
        EventId = 1001,
        Level = LogLevel.Information,
        Message = "Processing order {OrderId}")]
    private partial void LogProcessingOrder(OrderId orderId);

    [LoggerMessage(
        EventId = 1002,
        Level = LogLevel.Error,
        Message = "Failed to process order {OrderId}")]
    private partial void LogOrderFailed(Exception ex, OrderId orderId);

    public async Task ProcessOrder(OrderId orderId)
    {
        LogProcessingOrder(orderId);

        try
        {
            await ProcessOrderInternal(orderId);
        }
        catch (Exception ex)
        {
            LogOrderFailed(ex, orderId);
            throw;
        }
    }
}
```

## Symptoms

- Difficulty finding relevant logs in production
- Sensitive data exposed in logs
- Log files growing too large
- Performance degradation from excessive logging
- Lost exception details
- Inability to correlate related log entries

## Benefits

- **Searchable logs** with structured logging
- **Security** by excluding sensitive data
- **Better performance** by guarding expensive operations
- **Easier troubleshooting** with correlation IDs
- **Actionable insights** with semantic logging
- **Compliance** with data protection regulations

## See Also

- [Cross-Cutting Concerns](./type-safe-cross-cutting-concerns.md) — Type-safe logging
- [Correlation IDs](./correlation-ids.md) — Distributed tracing
- [Secret Types](./secret-types.md) — Preventing accidental exposure
