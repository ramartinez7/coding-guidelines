# Null Object Pattern

> Replace null references with objects that implement the expected interface but do nothing—eliminate null checks.

## Problem

Null checks scattered throughout code make it verbose and error-prone. Forgetting a single null check causes a `NullReferenceException` at runtime. The null object pattern provides a safe default behavior instead of null.

## Example

### ❌ Before

```csharp
public interface ILogger
{
    void Log(string message);
    void LogError(string message, Exception ex);
}

public class OrderService
{
    private readonly ILogger? logger;

    public OrderService(ILogger? logger)
    {
        this.logger = logger;
    }

    public void ProcessOrder(Order order)
    {
        // Null checks everywhere
        if (logger != null)
            logger.Log($"Processing order {order.Id}");
        
        try
        {
            // Business logic
            order.Process();
            
            if (logger != null)
                logger.Log($"Order {order.Id} processed successfully");
        }
        catch (Exception ex)
        {
            if (logger != null)
                logger.LogError($"Failed to process order {order.Id}", ex);
            
            throw;
        }
    }
}
```

### ✅ After

```csharp
public interface ILogger
{
    void Log(string message);
    void LogError(string message, Exception ex);
}

public sealed class ConsoleLogger : ILogger
{
    public void Log(string message)
    {
        Console.WriteLine($"[INFO] {message}");
    }

    public void LogError(string message, Exception ex)
    {
        Console.WriteLine($"[ERROR] {message}: {ex.Message}");
    }
}

public sealed class NullLogger : ILogger
{
    public static readonly NullLogger Instance = new();

    private NullLogger() { }

    public void Log(string message)
    {
        // Do nothing - intentionally empty
    }

    public void LogError(string message, Exception ex)
    {
        // Do nothing - intentionally empty
    }
}

public class OrderService
{
    private readonly ILogger logger;

    public OrderService(ILogger logger)
    {
        // Never null - use NullLogger.Instance if no logging needed
        this.logger = logger;
    }

    public void ProcessOrder(Order order)
    {
        // No null checks needed
        logger.Log($"Processing order {order.Id}");
        
        try
        {
            order.Process();
            logger.Log($"Order {order.Id} processed successfully");
        }
        catch (Exception ex)
        {
            logger.LogError($"Failed to process order {order.Id}", ex);
            throw;
        }
    }
}

// Usage
var service = new OrderService(NullLogger.Instance);  // No logging
var service2 = new OrderService(new ConsoleLogger()); // With logging
```

## Why It's a Problem

1. **Null checks everywhere**: Every use of a nullable dependency requires a null check.

2. **Easy to forget**: Missing one null check causes a runtime exception.

3. **Noise in business logic**: Null checks obscure the actual business logic.

4. **Inconsistent behavior**: Different parts of code may handle null differently.

5. **Testing complexity**: Must test both null and non-null cases.

## Benefits

- **Eliminates null checks**: Code is cleaner and easier to read
- **Safe by default**: No risk of `NullReferenceException`
- **Polymorphic**: Null object implements the same interface as real objects
- **Explicit intent**: `NullLogger.Instance` is clearer than `null`
- **Easier testing**: No special null handling in tests

## Common Use Cases

### 1. Logging

```csharp
public sealed class NullLogger : ILogger
{
    public static readonly NullLogger Instance = new();
    private NullLogger() { }
    
    public void Log(string message) { }
    public void LogError(string message, Exception ex) { }
}
```

### 2. Notifications

```csharp
public interface INotificationService
{
    Task NotifyAsync(UserId userId, string message);
}

public sealed class NullNotificationService : INotificationService
{
    public static readonly NullNotificationService Instance = new();
    private NullNotificationService() { }
    
    public Task NotifyAsync(UserId userId, string message)
    {
        return Task.CompletedTask;
    }
}
```

### 3. Caching

```csharp
public interface ICache
{
    T? Get<T>(string key);
    void Set<T>(string key, T value, TimeSpan expiration);
}

public sealed class NullCache : ICache
{
    public static readonly NullCache Instance = new();
    private NullCache() { }
    
    public T? Get<T>(string key) => default;
    
    public void Set<T>(string key, T value, TimeSpan expiration) { }
}
```

### 4. Collections

```csharp
public sealed class EmptyCustomerCollection : ICustomerCollection
{
    public static readonly EmptyCustomerCollection Instance = new();
    private EmptyCustomerCollection() { }
    
    public int Count => 0;
    
    public Customer? FindById(CustomerId id) => null;
    
    public IEnumerable<Customer> GetAll() => Enumerable.Empty<Customer>();
    
    public void Add(Customer customer)
    {
        throw new InvalidOperationException("Cannot add to empty collection");
    }
}
```

## Implementation Guidelines

### 1. Make It a Singleton

Use a static instance to avoid creating multiple null objects:

```csharp
public sealed class NullLogger : ILogger
{
    public static readonly NullLogger Instance = new();
    
    // Private constructor prevents external instantiation
    private NullLogger() { }
    
    public void Log(string message) { }
}
```

### 2. Document the Behavior

Make it clear that this is intentionally a no-op:

```csharp
/// <summary>
/// Null Object implementation that discards all log messages.
/// Use when logging is not required.
/// </summary>
public sealed class NullLogger : ILogger
{
    // ...
}
```

### 3. Consider the Interface

Some methods should return sensible defaults:

```csharp
public sealed class NullCache : ICache
{
    public T? Get<T>(string key)
    {
        // Return default value, not throw
        return default;
    }
    
    public bool Contains(string key)
    {
        // Always false makes sense
        return false;
    }
}
```

## When Not to Use

### 1. When Null Has Meaning

If null represents an important state, don't hide it:

```csharp
// ❌ Bad: Null has business meaning here
public interface IDiscount
{
    Money Calculate(Money price);
}

public class NullDiscount : IDiscount
{
    public Money Calculate(Money price) => Money.Zero;  // Wrong! No discount is different from zero discount
}

// ✅ Good: Use Option<T> or nullable type
public Option<Discount> GetApplicableDiscount(Customer customer);
```

### 2. When Operations Must Succeed

Don't use null objects for operations that should fail if unavailable:

```csharp
// ❌ Bad: Payment should fail if processor is unavailable
public sealed class NullPaymentProcessor : IPaymentProcessor
{
    public Task<PaymentResult> ProcessAsync(Payment payment)
    {
        return Task.FromResult(PaymentResult.Success());  // Lying about success!
    }
}

// ✅ Good: Fail explicitly
public sealed class PaymentService
{
    private readonly IPaymentProcessor? processor;
    
    public Result<PaymentResult, string> Process(Payment payment)
    {
        if (processor == null)
            return Result<PaymentResult, string>.Failure("Payment processor not configured");
        
        return processor.ProcessAsync(payment);
    }
}
```

### 3. With Nullable Reference Types

In modern C#, prefer nullable reference types over null objects for optional dependencies:

```csharp
// Modern approach with nullable reference types
public class OrderService
{
    private readonly ILogger? logger;

    public OrderService(ILogger? logger = null)
    {
        this.logger = logger;
    }

    public void ProcessOrder(Order order)
    {
        // Null-conditional operator is concise
        logger?.Log($"Processing order {order.Id}");
        
        // Business logic without null checks
        order.Process();
    }
}
```

## Null Object vs. Alternatives

| Pattern | Use When | Example |
|---------|----------|---------|
| **Null Object** | Dependency is truly optional and no-op is safe | Logging, telemetry, notifications |
| **Option\<T\>** | Absence of value has business meaning | Finding a customer by ID |
| **Nullable Types** | Value types that may be absent | Age, quantity (when 0 is different from absent) |
| **Result\<T, E\>** | Operation can fail with meaningful errors | Parsing, validation, external API calls |
| **Default Parameters** | Simple optional parameters | `void SendEmail(string to, string? cc = null)` |

## See Also

- [Nullability vs. Optionality](./nullability-optionality.md) — when to use Option\<T\> vs null
- [Nullable Reference Types](./nullable-reference-types.md) — compiler-enforced null safety
- [Honest Functions](./honest-functions.md) — explicit error handling with Result types
- [Dependency Injection](./dependency-injection.md) — injecting dependencies including null objects
