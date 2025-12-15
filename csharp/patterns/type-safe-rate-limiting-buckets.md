# Type-Safe Rate Limiting Buckets

> Rate limiting with counters and timestamps scattered in code—use typed rate limit buckets to enforce rate limits through the type system.

## Problem

Rate limiting implementations using in-memory counters or distributed caches with string keys allow rate limit bypass, incorrect bucket selection, and missing rate limit checks. The compiler can't verify that rate limits are correctly applied.

## Example

### ❌ Before

```csharp
public class ApiController
{
    public async Task<IActionResult> GetData(string userId)
    {
        var key = $"ratelimit:{userId}";
        var count = await _cache.GetIntAsync(key);
        
        if (count >= 100)
            return StatusCode(429);
        
        await _cache.IncrementAsync(key);
        return Ok(await GetDataInternal());
    }
}
```

### ✅ After

```csharp
public interface IRateLimitPolicy
{
    int MaxRequests { get; }
    TimeSpan Window { get; }
}

public sealed record UserApiRateLimit : IRateLimitPolicy
{
    public int MaxRequests => 100;
    public TimeSpan Window => TimeSpan.FromMinutes(1);
}

public sealed record AdminApiRateLimit : IRateLimitPolicy
{
    public int MaxRequests => 1000;
    public TimeSpan Window => TimeSpan.FromMinutes(1);
}

public sealed class RateLimitBucket<TPolicy> where TPolicy : IRateLimitPolicy, new()
{
    private readonly string _identifier;
    private readonly TPolicy _policy = new();
    
    public RateLimitBucket(string identifier)
    {
        _identifier = identifier;
    }
    
    public string Key => $"ratelimit:{typeof(TPolicy).Name}:{_identifier}";
    public IRateLimitPolicy Policy => _policy;
}

public interface IRateLimiter
{
    Task<Result<Unit, string>> CheckAsync<TPolicy>(RateLimitBucket<TPolicy> bucket)
        where TPolicy : IRateLimitPolicy, new();
}

public sealed class RateLimiter : IRateLimiter
{
    private readonly IDistributedCache _cache;
    
    public async Task<Result<Unit, string>> CheckAsync<TPolicy>(
        RateLimitBucket<TPolicy> bucket)
        where TPolicy : IRateLimitPolicy, new()
    {
        var count = await _cache.GetAsync<int>(bucket.Key);
        
        if (count >= bucket.Policy.MaxRequests)
            return Result<Unit, string>.Failure("Rate limit exceeded");
        
        await _cache.IncrementAsync(bucket.Key, bucket.Policy.Window);
        return Result<Unit, string>.Success(Unit.Value);
    }
}

// Usage
public class ApiController
{
    public async Task<IActionResult> GetData(UserId userId)
    {
        var bucket = new RateLimitBucket<UserApiRateLimit>(userId.Value.ToString());
        var result = await _rateLimiter.CheckAsync(bucket);
        
        return result.Match(
            _ => Ok(await GetDataInternal()),
            error => StatusCode(429, error));
    }
}
```

## See Also

- [Rate Limiting Tokens](./rate-limiting-tokens.md)
- [Phantom Types](./phantom-types.md)
