# Type-Safe Health Checks (Dependency Health Encoding)

> Health check endpoints with string-based status and no type safety—use typed health descriptors to make health check contracts explicit and composable.

## Problem

Health check implementations return generic `HealthCheckResult` objects with string descriptions and tags. Critical dependencies can fail silently, health status lacks type safety, and aggregating health checks requires runtime string parsing.

## Example

### ❌ Before

```csharp
public class DatabaseHealthCheck : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            await _db.Database.CanConnectAsync(cancellationToken);
            return HealthCheckResult.Healthy("Database is reachable");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database is down", ex);
        }
    }
}
```

### ✅ After

```csharp
public interface IHealthStatus { }
public interface IHealthy : IHealthStatus { }
public interface IDegraded : IHealthStatus { }
public interface IUnhealthy : IHealthStatus { }

public sealed record HealthCheck<TDependency, TStatus>
    where TStatus : IHealthStatus
{
    public string DependencyName { get; }
    public TStatus Status { get; }
    public Option<string> Message { get; }
    public Option<Exception> Exception { get; }
    
    private HealthCheck(
        string dependencyName,
        TStatus status,
        Option<string> message,
        Option<Exception> exception)
    {
        DependencyName = dependencyName;
        Status = status;
        Message = message;
        Exception = exception;
    }
    
    public static HealthCheck<TDependency, IHealthy> Healthy(
        string message = "")
    {
        return new HealthCheck<TDependency, IHealthy>(
            typeof(TDependency).Name,
            default(IHealthy)!,
            string.IsNullOrEmpty(message) 
                ? Option<string>.None() 
                : Option<string>.Some(message),
            Option<Exception>.None());
    }
    
    public static HealthCheck<TDependency, IUnhealthy> Unhealthy(
        string message,
        Exception? exception = null)
    {
        return new HealthCheck<TDependency, IUnhealthy>(
            typeof(TDependency).Name,
            default(IUnhealthy)!,
            Option<string>.Some(message),
            exception is null 
                ? Option<Exception>.None() 
                : Option<Exception>.Some(exception));
    }
}

public interface IDatabase { }
public interface ICache { }
public interface IMessageQueue { }

public sealed class TypedHealthCheckService
{
    public async Task<HealthCheck<IDatabase, IHealthStatus>> CheckDatabaseAsync()
    {
        try
        {
            await _db.Database.CanConnectAsync();
            return HealthCheck<IDatabase, IHealthy>.Healthy("Connected");
        }
        catch (Exception ex)
        {
            return HealthCheck<IDatabase, IUnhealthy>.Unhealthy(
                "Cannot connect",
                ex);
        }
    }
    
    public async Task<HealthCheck<ICache, IHealthStatus>> CheckCacheAsync()
    {
        try
        {
            await _cache.GetAsync("health-check");
            return HealthCheck<ICache, IHealthy>.Healthy("Responding");
        }
        catch (Exception ex)
        {
            return HealthCheck<ICache, IUnhealthy>.Unhealthy(
                "Not responding",
                ex);
        }
    }
}
```

## See Also

- [Discriminated Unions](./discriminated-unions.md)
- [Phantom Types](./phantom-types.md)
