# Type-Safe Aspect-Oriented Composition

> Cross-cutting concerns applied with reflection or dynamic proxies fail silently—use typed decorators and composition to make aspects compile-time safe.

## Problem

Aspect-oriented programming (AOP) frameworks using attribute-based or proxy-based interception operate at runtime through reflection. Aspects can fail to apply, have incorrect ordering, or break when method signatures change. The compiler can't verify aspect composition.

## Example

### ❌ Before

```csharp
public class UserService
{
    [Transaction]
    [Logging]
    [Caching]
    public async Task<User> GetUserAsync(int id)
    {
        return await _repository.GetByIdAsync(id);
    }
}

// Problems: Attributes applied at runtime, no compile-time verification
```

### ✅ After

```csharp
public interface IUserService
{
    Task<User> GetUserAsync(UserId id);
}

public sealed class UserService : IUserService
{
    public async Task<User> GetUserAsync(UserId id)
    {
        return await _repository.GetByIdAsync(id);
    }
}

public sealed class LoggingDecorator<T> : IUserService where T : IUserService
{
    private readonly T _inner;
    private readonly ILogger _logger;
    
    public LoggingDecorator(T inner, ILogger logger)
    {
        _inner = inner;
        _logger = logger;
    }
    
    public async Task<User> GetUserAsync(UserId id)
    {
        _logger.LogInformation("Getting user {UserId}", id);
        var result = await _inner.GetUserAsync(id);
        _logger.LogInformation("Retrieved user {UserId}", id);
        return result;
    }
}

public sealed class CachingDecorator<T> : IUserService where T : IUserService
{
    private readonly T _inner;
    private readonly ICache _cache;
    
    public CachingDecorator(T inner, ICache cache)
    {
        _inner = inner;
        _cache = cache;
    }
    
    public async Task<User> GetUserAsync(UserId id)
    {
        var key = $"user:{id}";
        var cached = await _cache.GetAsync<User>(key);
        
        if (cached.IsSome)
            return cached.Value;
        
        var user = await _inner.GetUserAsync(id);
        await _cache.SetAsync(key, user);
        return user;
    }
}

// Type-safe composition
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddScoped<UserService>();
        services.AddScoped<IUserService>(sp =>
        {
            var core = sp.GetRequiredService<UserService>();
            var withLogging = new LoggingDecorator<UserService>(
                core,
                sp.GetRequiredService<ILogger>());
            var withCaching = new CachingDecorator<LoggingDecorator<UserService>>(
                withLogging,
                sp.GetRequiredService<ICache>());
            
            return withCaching;
        });
    }
}
```

## See Also

- [Type-Safe Dependency Injection](./type-safe-dependency-injection.md)
- [Dependency Injection](./dependency-injection.md)
