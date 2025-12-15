# Type-Safe Audit Trail (Event Type Safety)

> Audit logs with string messages and untyped data lose type information—use typed audit events to make audit trails queryable and type-safe.

## Problem

Audit logging systems that record events as strings or key-value dictionaries lose type information, making querying difficult and preventing compile-time verification of logged data. Critical audit information can be missing or incorrectly formatted.

## Example

### ❌ Before

```csharp
public class AuditLogger
{
    public void Log(string userId, string action, Dictionary<string, object> data)
    {
        var entry = new
        {
            UserId = userId,
            Action = action,
            Data = data,
            Timestamp = DateTime.UtcNow
        };
        _db.AuditLog.Add(entry);
    }
}

// Usage: No type safety
_logger.Log("user123", "OrderCreated", new Dictionary<string, object>
{
    ["OrderId"] = orderId.ToString(),  // Typo in key?
    ["Amount"] = amount.ToString()  // Should be decimal, not string?
});
```

### ✅ After

```csharp
public interface IAuditEvent
{
    string EventType { get; }
    DateTimeOffset Timestamp { get; }
}

public sealed record OrderCreatedEvent : IAuditEvent
{
    public required UserId UserId { get; init; }
    public required OrderId OrderId { get; init; }
    public required Money Amount { get; init; }
    
    public string EventType => "OrderCreated";
    public DateTimeOffset Timestamp { get; init; } = DateTimeOffset.UtcNow;
}

public sealed record UserLoggedInEvent : IAuditEvent
{
    public required UserId UserId { get; init; }
    public required IPAddress IpAddress { get; init; }
    
    public string EventType => "UserLoggedIn";
    public DateTimeOffset Timestamp { get; init; } = DateTimeOffset.UtcNow;
}

public interface IAuditLogger
{
    Task LogAsync<TEvent>(TEvent auditEvent) where TEvent : IAuditEvent;
    Task<IReadOnlyList<TEvent>> QueryAsync<TEvent>() where TEvent : IAuditEvent;
}

public sealed class TypedAuditLogger : IAuditLogger
{
    private readonly DbContext _db;
    
    public async Task LogAsync<TEvent>(TEvent auditEvent) where TEvent : IAuditEvent
    {
        var json = JsonSerializer.Serialize(auditEvent);
        var entry = new AuditLogEntry
        {
            EventType = auditEvent.EventType,
            Timestamp = auditEvent.Timestamp,
            Data = json
        };
        
        _db.AuditLog.Add(entry);
        await _db.SaveChangesAsync();
    }
    
    public async Task<IReadOnlyList<TEvent>> QueryAsync<TEvent>() where TEvent : IAuditEvent
    {
        var eventType = typeof(TEvent).Name.Replace("Event", "");
        
        var entries = await _db.AuditLog
            .Where(e => e.EventType == eventType)
            .ToListAsync();
        
        return entries
            .Select(e => JsonSerializer.Deserialize<TEvent>(e.Data)!)
            .ToList();
    }
}

// Usage: Type-safe audit logging
await _auditLogger.LogAsync(new OrderCreatedEvent
{
    UserId = userId,
    OrderId = orderId,
    Amount = amount
});

// Type-safe querying
var loginEvents = await _auditLogger.QueryAsync<UserLoggedInEvent>();
```

## See Also

- [Domain Events](./domain-events.md)
- [Type-Safe Messaging](./type-safe-messaging.md)
- [Strongly Typed IDs](./strongly-typed-ids.md)
