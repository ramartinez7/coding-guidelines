# Type-Safe Background Job Scheduling

> Background jobs scheduled with string job names and untyped parameters fail silently—use typed job descriptors to make job scheduling compile-time safe.

## Problem

Background job frameworks that accept job names as strings and parameters as objects allow typos in job names, missing parameters, and type mismatches. Jobs can fail at runtime when parameters are incorrect or when job handlers are renamed or removed.

## Example

### ❌ Before

```csharp
public class OrderService
{
    public async Task CreateOrderAsync(Order order)
    {
        await _repository.SaveAsync(order);
        
        // String-based job scheduling
        _jobScheduler.Enqueue("SendOrderConfirmation", new { orderId = order.Id });
        _jobScheduler.Schedule("SendFollowUpEmail", TimeSpan.FromDays(7), 
            new { orderId = order.Id });
    }
}

public class EmailJobs
{
    public void SendOrderConfirmation(int orderId)  // Type mismatch if Guid!
    {
        // Send email
    }
}
```

### ✅ After

```csharp
public interface IBackgroundJob<TParameters>
{
    Task ExecuteAsync(TParameters parameters);
}

public sealed record SendOrderConfirmationParameters(OrderId OrderId, Email CustomerEmail);

public sealed class SendOrderConfirmationJob 
    : IBackgroundJob<SendOrderConfirmationParameters>
{
    private readonly IEmailService _emailService;
    
    public async Task ExecuteAsync(SendOrderConfirmationParameters parameters)
    {
        await _emailService.SendAsync(
            parameters.CustomerEmail,
            "Order Confirmation",
            $"Your order {parameters.OrderId} has been confirmed");
    }
}

public sealed record ScheduledJob<TParameters>(
    IBackgroundJob<TParameters> Job,
    TParameters Parameters,
    Option<TimeSpan> Delay);

public interface ITypedJobScheduler
{
    Task EnqueueAsync<TJob, TParameters>(TParameters parameters)
        where TJob : IBackgroundJob<TParameters>, new();
    
    Task ScheduleAsync<TJob, TParameters>(TParameters parameters, TimeSpan delay)
        where TJob : IBackgroundJob<TParameters>, new();
}

public sealed class TypedJobScheduler : ITypedJobScheduler
{
    private readonly IBackgroundJobClient _client;
    
    public Task EnqueueAsync<TJob, TParameters>(TParameters parameters)
        where TJob : IBackgroundJob<TParameters>, new()
    {
        var job = new TJob();
        return _client.Enqueue(() => job.ExecuteAsync(parameters));
    }
    
    public Task ScheduleAsync<TJob, TParameters>(TParameters parameters, TimeSpan delay)
        where TJob : IBackgroundJob<TParameters>, new()
    {
        var job = new TJob();
        return _client.Schedule(() => job.ExecuteAsync(parameters), delay);
    }
}

// Usage: Type-safe job scheduling
public class OrderService
{
    private readonly ITypedJobScheduler _jobScheduler;
    
    public async Task CreateOrderAsync(Order order)
    {
        await _repository.SaveAsync(order);
        
        // Compile-time type safety
        await _jobScheduler.EnqueueAsync<SendOrderConfirmationJob, SendOrderConfirmationParameters>(
            new SendOrderConfirmationParameters(order.Id, order.CustomerEmail));
        
        await _jobScheduler.ScheduleAsync<SendFollowUpEmailJob, SendFollowUpParameters>(
            new SendFollowUpParameters(order.Id),
            TimeSpan.FromDays(7));
    }
}

// Won't compile:
// _jobScheduler.EnqueueAsync<SendOrderConfirmationJob, WrongParameters>(...)
// Error: Type mismatch
```

## See Also

- [Type-Safe Messaging](./type-safe-messaging.md)
- [Strongly Typed IDs](./strongly-typed-ids.md)
- [Smart Constructors](./smart-constructors.md)
