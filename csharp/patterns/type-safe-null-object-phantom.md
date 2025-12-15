# Type-Safe Null Object with Phantom Types

> Null object pattern implemented with single class lacks type information—use phantom types to distinguish between null objects and real instances at compile time.

## Problem

Traditional null object pattern returns objects that implement an interface but do nothing. Callers can't distinguish between null objects and real instances without runtime checks, leading to logic errors when null objects are used where real instances are expected.

## Example

### ❌ Before

```csharp
public interface INotificationService
{
    Task SendAsync(string message);
}

public class EmailNotificationService : INotificationService
{
    public async Task SendAsync(string message)
    {
        await _smtp.SendAsync(message);
    }
}

public class NullNotificationService : INotificationService
{
    public Task SendAsync(string message) => Task.CompletedTask;
}

// Problem: Can't distinguish at compile time
INotificationService service = GetService();  // Real or null?
```

### ✅ After

```csharp
public interface IServiceState { }
public interface IReal : IServiceState { }
public interface INullObject : IServiceState { }

public sealed class NotificationService<TState> where TState : IServiceState
{
    private readonly IEmailSender? _sender;
    
    private NotificationService(IEmailSender? sender)
    {
        _sender = sender;
    }
    
    public static NotificationService<IReal> CreateReal(IEmailSender sender)
        => new(sender);
    
    public static NotificationService<INullObject> CreateNull()
        => new(null);
}

public static class NotificationServiceExtensions
{
    public static async Task SendAsync(
        this NotificationService<IReal> service,
        string message)
    {
        await service._sender!.SendAsync(message);
    }
    
    // Null object version does nothing—but type is explicit
    public static Task SendAsync(
        this NotificationService<INullObject> service,
        string message)
    {
        return Task.CompletedTask;
    }
}

// Usage: Type makes null/real explicit
NotificationService<IReal> realService = NotificationService<IReal>.CreateReal(sender);
NotificationService<INullObject> nullService = NotificationService<INullObject>.CreateNull();
```

## See Also

- [Null Object Pattern](./null-object-pattern.md)
- [Phantom Types](./phantom-types.md)
- [Option Monad](./option-monad.md)
