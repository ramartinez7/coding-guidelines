# Type-Safe Cross-Cutting Concerns (Logging and Metrics)

> Logging and metrics with string keys and untyped values lose type safety—use typed telemetry to make cross-cutting concerns compile-time safe.

## Problem

Logging frameworks that accept string messages and untyped properties allow typos in metric names, missing required fields, and incompatible value types. Telemetry code is scattered throughout the application with no compile-time validation.

## Example

### ❌ Before

```csharp
public class OrderService
{
    public async Task CreateOrderAsync(Order order)
    {
        _logger.LogInformation("Order created: {OrderId}", order.Id);
        _metrics.Increment("orders_created");  // String key—typo risk
        
        await _repository.SaveAsync(order);
        
        _logger.LogInformation("Order saved: {OrderID}", order.Id);  // Different casing!
    }
}
```

### ✅ After

```csharp
public interface ILogEvent
{
    string Template { get; }
}

public sealed record OrderCreatedLog(OrderId OrderId) : ILogEvent
{
    public string Template => "Order created: {OrderId}";
}

public sealed record OrderSavedLog(OrderId OrderId, Money Amount) : ILogEvent
{
    public string Template => "Order saved: {OrderId} for {Amount}";
}

public interface IMetric
{
    string Name { get; }
}

public sealed record OrdersCreatedMetric : IMetric
{
    public string Name => "orders.created";
}

public interface ITypedLogger
{
    void Log<TEvent>(TEvent logEvent) where TEvent : ILogEvent;
}

public interface ITypedMetrics
{
    void Increment<TMetric>() where TMetric : IMetric, new();
    void Increment<TMetric>(TMetric metric) where TMetric : IMetric;
}

public sealed class TypedLogger : ITypedLogger
{
    private readonly ILogger _logger;
    
    public void Log<TEvent>(TEvent logEvent) where TEvent : ILogEvent
    {
        var properties = typeof(TEvent)
            .GetProperties()
            .Where(p => p.Name != nameof(ILogEvent.Template))
            .Select(p => p.GetValue(logEvent))
            .ToArray();
        
        _logger.LogInformation(logEvent.Template, properties);
    }
}

// Usage: Type-safe logging
public class OrderService
{
    private readonly ITypedLogger _logger;
    private readonly ITypedMetrics _metrics;
    
    public async Task CreateOrderAsync(Order order)
    {
        _logger.Log(new OrderCreatedLog(order.Id));
        _metrics.Increment<OrdersCreatedMetric>();
        
        await _repository.SaveAsync(order);
        
        _logger.Log(new OrderSavedLog(order.Id, order.Total));
    }
}
```

## See Also

- [Type-Safe Audit Trail](./type-safe-audit-trail.md)
- [Strongly Typed IDs](./strongly-typed-ids.md)
