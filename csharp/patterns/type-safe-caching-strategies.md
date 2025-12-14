# Type-Safe Caching Strategies (Cache Key Safety)

> String-based cache keys collide, expire silently, and can't be invalidated reliably—use typed cache keys to prevent cache pollution and ensure type-safe cache operations.

## Problem

Caching mechanisms using string keys allow key collisions, make invalidation error-prone, and provide no compile-time guarantees about cached value types. Keys can be misspelled, prefixes forgotten, or parameters omitted.

## Example

### ❌ Before

```csharp
public class UserService
{
    private readonly IDistributedCache _cache;
    
    public async Task<User> GetUserAsync(int id)
    {
        var key = $"user:{id}";  // String construction—fragile
        var cached = await _cache.GetStringAsync(key);
        
        if (cached != null)
            return JsonSerializer.Deserialize<User>(cached)!;
        
        var user = await _db.Users.FindAsync(id);
        await _cache.SetStringAsync(key, JsonSerializer.Serialize(user));
        return user;
    }
    
    public async Task InvalidateUserAsync(int id)
    {
        await _cache.RemoveAsync($"usr:{id}");  // Typo—wrong key!
    }
}
```

### ✅ After

```csharp
public interface ICacheKey<T>
{
    string Key { get; }
    TimeSpan? Expiration { get; }
}

public sealed record UserCacheKey(UserId Id) : ICacheKey<User>
{
    public string Key => $"user:{Id.Value}";
    public TimeSpan? Expiration => TimeSpan.FromMinutes(30);
}

public sealed record OrderCacheKey(OrderId Id) : ICacheKey<Order>
{
    public string Key => $"order:{Id.Value}";
    public TimeSpan? Expiration => TimeSpan.FromHours(1);
}

public interface ITypedCache
{
    Task<Option<T>> GetAsync<T>(ICacheKey<T> key);
    Task SetAsync<T>(ICacheKey<T> key, T value);
    Task RemoveAsync<T>(ICacheKey<T> key);
}

public sealed class TypedCache : ITypedCache
{
    private readonly IDistributedCache _cache;
    
    public TypedCache(IDistributedCache cache)
    {
        _cache = cache;
    }
    
    public async Task<Option<T>> GetAsync<T>(ICacheKey<T> key)
    {
        var cached = await _cache.GetStringAsync(key.Key);
        if (cached is null)
            return Option<T>.None();
        
        var value = JsonSerializer.Deserialize<T>(cached);
        return value is null
            ? Option<T>.None()
            : Option<T>.Some(value);
    }
    
    public async Task SetAsync<T>(ICacheKey<T> key, T value)
    {
        var json = JsonSerializer.Serialize(value);
        var options = new DistributedCacheEntryOptions();
        
        if (key.Expiration.HasValue)
            options.AbsoluteExpirationRelativeToNow = key.Expiration;
        
        await _cache.SetStringAsync(key.Key, json, options);
    }
    
    public async Task RemoveAsync<T>(ICacheKey<T> key)
    {
        await _cache.RemoveAsync(key.Key);
    }
}

// Usage: Type-safe caching
public class UserService
{
    private readonly ITypedCache _cache;
    
    public async Task<User> GetUserAsync(UserId id)
    {
        var key = new UserCacheKey(id);
        var cached = await _cache.GetAsync(key);
        
        if (cached.IsSome)
            return cached.Value;
        
        var user = await _db.Users.FindAsync(id);
        await _cache.SetAsync(key, user);
        return user;
    }
    
    public async Task InvalidateUserAsync(UserId id)
    {
        await _cache.RemoveAsync(new UserCacheKey(id));
    }
}

// Won't compile:
// await _cache.SetAsync(new UserCacheKey(id), order);  // Error: type mismatch
```

## See Also

- [Strongly Typed IDs](./strongly-typed-ids.md)
- [Type-Safe Configuration](./type-safe-configuration.md)
