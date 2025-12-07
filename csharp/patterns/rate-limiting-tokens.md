# Rate Limiting (Type-Safe Token Bucket)

> Manual rate limit checks scattered throughout code—use typed rate limit tokens to enforce limits at compile time.

## Problem

Traditional rate limiting relies on middleware that returns HTTP 429 responses, but this happens too late—after authentication, authorization, and potentially expensive business logic. The caller must handle retry logic manually, and there's no compile-time guarantee that rate limits are enforced consistently.

## Example

### ❌ Before

```csharp
[ApiController]
[Route("api/search")]
public class SearchController : ControllerBase
{
    private readonly IRateLimiter _rateLimiter;
    private readonly ISearchService _searchService;
    
    [HttpGet]
    public async Task<IActionResult> Search(string query)
    {
        var clientId = GetClientId();
        
        // Runtime check—happens after routing, binding, etc.
        if (!await _rateLimiter.AllowRequest(clientId, "search"))
        {
            return StatusCode(429, "Too many requests");
        }
        
        // Expensive operation happens after we checked
        var results = await _searchService.SearchAsync(query);
        return Ok(results);
    }
}

// No type safety—easy to forget rate limit check
[HttpPost("expensive-operation")]
public async Task<IActionResult> ExpensiveOperation([FromBody] Request request)
{
    // Oops! Forgot to check rate limit
    var result = await _service.DoExpensiveWork(request);
    return Ok(result);
}
```

**Problems:**
- Rate limit checked after expensive routing/binding
- Easy to forget rate limit check in new endpoints
- No compile-time guarantee
- Client retry logic not coordinated with server
- No way to "spend" different amounts for different operations

### ✅ After

```csharp
/// <summary>
/// Proof that rate limit has been checked and request is allowed.
/// Cannot be constructed directly—must come from rate limiter.
/// </summary>
public sealed record RateLimitToken
{
    public required ClientId ClientId { get; init; }
    public required DateTime GrantedAt { get; init; }
    public required DateTime ExpiresAt { get; init; }
    public required int RemainingTokens { get; init; }
    
    internal RateLimitToken() { }  // Force construction through factory
}

/// <summary>
/// Result of attempting to acquire a rate limit token.
/// </summary>
public abstract record RateLimitResult
{
    private RateLimitResult() { }
    
    public sealed record Allowed(RateLimitToken Token) : RateLimitResult;
    
    public sealed record Denied(
        int RemainingTokens,
        DateTime RetryAfter,
        string Reason) : RateLimitResult;
}

/// <summary>
/// Rate limiter that issues unforgeable tokens.
/// </summary>
public interface IRateLimiter
{
    Task<RateLimitResult> TryAcquireAsync(ClientId clientId, int cost = 1);
}

// Token bucket implementation
public class TokenBucketRateLimiter : IRateLimiter
{
    private readonly int _capacity;
    private readonly int _refillRate;  // tokens per second
    private readonly Dictionary<ClientId, Bucket> _buckets = new();
    
    public TokenBucketRateLimiter(int capacity, int refillRate)
    {
        _capacity = capacity;
        _refillRate = refillRate;
    }
    
    public async Task<RateLimitResult> TryAcquireAsync(ClientId clientId, int cost = 1)
    {
        var bucket = GetOrCreateBucket(clientId);
        
        await bucket.RefillAsync(_refillRate);
        
        if (bucket.Available >= cost)
        {
            bucket.Consume(cost);
            
            return new RateLimitResult.Allowed(new RateLimitToken
            {
                ClientId = clientId,
                GrantedAt = DateTime.UtcNow,
                ExpiresAt = DateTime.UtcNow.AddSeconds(60),
                RemainingTokens = bucket.Available
            });
        }
        
        var retryAfter = bucket.EstimateRefillTime(cost);
        
        return new RateLimitResult.Denied(
            RemainingTokens: bucket.Available,
            RetryAfter: retryAfter,
            Reason: $"Rate limit exceeded. Need {cost} tokens, have {bucket.Available}");
    }
    
    private Bucket GetOrCreateBucket(ClientId clientId)
    {
        if (!_buckets.TryGetValue(clientId, out var bucket))
        {
            bucket = new Bucket(_capacity);
            _buckets[clientId] = bucket;
        }
        return bucket;
    }
    
    private class Bucket
    {
        private int _tokens;
        private DateTime _lastRefill;
        private readonly int _capacity;
        
        public int Available => _tokens;
        
        public Bucket(int capacity)
        {
            _capacity = capacity;
            _tokens = capacity;
            _lastRefill = DateTime.UtcNow;
        }
        
        public Task RefillAsync(int refillRate)
        {
            var now = DateTime.UtcNow;
            var elapsed = (now - _lastRefill).TotalSeconds;
            var tokensToAdd = (int)(elapsed * refillRate);
            
            if (tokensToAdd > 0)
            {
                _tokens = Math.Min(_capacity, _tokens + tokensToAdd);
                _lastRefill = now;
            }
            
            return Task.CompletedTask;
        }
        
        public void Consume(int count)
        {
            _tokens = Math.Max(0, _tokens - count);
        }
        
        public DateTime EstimateRefillTime(int needed)
        {
            var deficit = needed - _tokens;
            if (deficit <= 0)
                return DateTime.UtcNow;
            
            return DateTime.UtcNow.AddSeconds(deficit);
        }
    }
}

/// <summary>
/// Strongly typed client identifier for rate limiting.
/// </summary>
public readonly record struct ClientId
{
    public string Value { get; }
    
    private ClientId(string value) => Value = value;
    
    public static ClientId FromApiKey(string apiKey)
    {
        if (string.IsNullOrWhiteSpace(apiKey))
            throw new ArgumentException("API key cannot be empty");
        return new ClientId(apiKey);
    }
    
    public static ClientId FromIpAddress(string ipAddress)
    {
        if (string.IsNullOrWhiteSpace(ipAddress))
            throw new ArgumentException("IP address cannot be empty");
        return new ClientId($"ip:{ipAddress}");
    }
    
    public static ClientId FromUserId(Guid userId)
    {
        return new ClientId($"user:{userId}");
    }
}

// Service methods require rate limit token as proof
public interface ISearchService
{
    Task<SearchResults> SearchAsync(
        RateLimitToken rateLimitToken,  // Proof required!
        string query);
}

public class SearchService : ISearchService
{
    public async Task<SearchResults> SearchAsync(
        RateLimitToken rateLimitToken,
        string query)
    {
        // Type system guarantees we can't be called without a token
        // Token proves rate limit was checked and passed
        
        var results = await PerformExpensiveSearchAsync(query);
        return results;
    }
}

// Controller must acquire token before calling service
[ApiController]
[Route("api/search")]
public class SearchController : ControllerBase
{
    private readonly IRateLimiter _rateLimiter;
    private readonly ISearchService _searchService;
    
    [HttpGet]
    public async Task<IActionResult> Search(string query)
    {
        var clientId = ClientId.FromIpAddress(
            HttpContext.Connection.RemoteIpAddress?.ToString() ?? "unknown");
        
        // Acquire token (cheap operation)
        var result = await _rateLimiter.TryAcquireAsync(clientId, cost: 1);
        
        return result switch
        {
            RateLimitResult.Allowed allowed =>
                await HandleAllowedRequest(allowed.Token, query),
                
            RateLimitResult.Denied denied =>
                RateLimitExceeded(denied),
                
            _ => throw new InvalidOperationException("Unknown rate limit result")
        };
    }
    
    private async Task<IActionResult> HandleAllowedRequest(
        RateLimitToken token,
        string query)
    {
        // Token is proof we can proceed—service enforces this
        var results = await _searchService.SearchAsync(token, query);
        
        // Include rate limit info in response headers
        Response.Headers.Add("X-RateLimit-Remaining", 
            token.RemainingTokens.ToString());
        Response.Headers.Add("X-RateLimit-Reset", 
            token.ExpiresAt.ToString("R"));
        
        return Ok(results);
    }
    
    private IActionResult RateLimitExceeded(RateLimitResult.Denied denied)
    {
        Response.Headers.Add("X-RateLimit-Remaining", "0");
        Response.Headers.Add("Retry-After", 
            ((int)(denied.RetryAfter - DateTime.UtcNow).TotalSeconds).ToString());
        
        return StatusCode(429, new
        {
            error = "rate_limit_exceeded",
            message = denied.Reason,
            retryAfter = denied.RetryAfter
        });
    }
}
```

## Different Costs for Different Operations

```csharp
// Expensive operations cost more tokens
[HttpPost("complex-search")]
public async Task<IActionResult> ComplexSearch([FromBody] ComplexQuery query)
{
    var clientId = GetClientId();
    
    // Complex search costs 10 tokens
    var result = await _rateLimiter.TryAcquireAsync(clientId, cost: 10);
    
    return result switch
    {
        RateLimitResult.Allowed allowed =>
            await HandleComplexSearch(allowed.Token, query),
        RateLimitResult.Denied denied =>
            RateLimitExceeded(denied),
        _ => throw new InvalidOperationException()
    };
}

// Cheap operations cost less
[HttpGet("health")]
public IActionResult Health()
{
    // Health checks are free—no token required
    return Ok(new { status = "healthy" });
}
```

## Hierarchical Rate Limits

```csharp
/// <summary>
/// Rate limiter with multiple tiers.
/// </summary>
public class HierarchicalRateLimiter : IRateLimiter
{
    private readonly IRateLimiter _perSecond;
    private readonly IRateLimiter _perMinute;
    private readonly IRateLimiter _perHour;
    
    public HierarchicalRateLimiter()
    {
        _perSecond = new TokenBucketRateLimiter(capacity: 10, refillRate: 10);
        _perMinute = new TokenBucketRateLimiter(capacity: 100, refillRate: 100);
        _perHour = new TokenBucketRateLimiter(capacity: 1000, refillRate: 1000);
    }
    
    public async Task<RateLimitResult> TryAcquireAsync(ClientId clientId, int cost = 1)
    {
        // Check all tiers—must pass all
        var secondResult = await _perSecond.TryAcquireAsync(clientId, cost);
        if (secondResult is RateLimitResult.Denied)
            return secondResult;
        
        var minuteResult = await _perMinute.TryAcquireAsync(clientId, cost);
        if (minuteResult is RateLimitResult.Denied)
            return minuteResult;
        
        var hourResult = await _perHour.TryAcquireAsync(clientId, cost);
        return hourResult;
    }
}
```

## Quota System

```csharp
/// <summary>
/// Monthly quota token—different from rate limit token.
/// </summary>
public sealed record QuotaToken
{
    public required ClientId ClientId { get; init; }
    public required int QuotaUsed { get; init; }
    public required int QuotaLimit { get; init; }
    public required DateTime ResetDate { get; init; }
    
    internal QuotaToken() { }
}

public abstract record QuotaResult
{
    private QuotaResult() { }
    
    public sealed record Available(QuotaToken Token) : QuotaResult;
    public sealed record Exceeded(int Used, int Limit, DateTime ResetDate) : QuotaResult;
}

public interface IQuotaManager
{
    Task<QuotaResult> TryConsumeAsync(ClientId clientId, int units = 1);
}

// Service requires BOTH rate limit token AND quota token
public interface IPremiumSearchService
{
    Task<SearchResults> SearchAsync(
        RateLimitToken rateLimitToken,
        QuotaToken quotaToken,
        string query);
}
```

## Testing

```csharp
public class RateLimiterTests
{
    [Fact]
    public async Task TryAcquire_WithinLimit_ReturnsToken()
    {
        var limiter = new TokenBucketRateLimiter(capacity: 10, refillRate: 1);
        var clientId = ClientId.FromApiKey("test-key");
        
        var result = await limiter.TryAcquireAsync(clientId);
        
        Assert.IsType<RateLimitResult.Allowed>(result);
        var allowed = (RateLimitResult.Allowed)result;
        Assert.Equal(clientId, allowed.Token.ClientId);
    }
    
    [Fact]
    public async Task TryAcquire_ExceedingLimit_ReturnsDenied()
    {
        var limiter = new TokenBucketRateLimiter(capacity: 2, refillRate: 1);
        var clientId = ClientId.FromApiKey("test-key");
        
        // Consume all tokens
        await limiter.TryAcquireAsync(clientId);
        await limiter.TryAcquireAsync(clientId);
        
        // Next request should be denied
        var result = await limiter.TryAcquireAsync(clientId);
        
        Assert.IsType<RateLimitResult.Denied>(result);
        var denied = (RateLimitResult.Denied)result;
        Assert.True(denied.RetryAfter > DateTime.UtcNow);
    }
    
    [Fact]
    public async Task Service_WithoutToken_DoesNotCompile()
    {
        var service = new SearchService();
        
        // This won't compile—token required!
        // var results = await service.SearchAsync("query");
        
        // Must have a token
        var limiter = new TokenBucketRateLimiter(10, 1);
        var result = await limiter.TryAcquireAsync(ClientId.FromApiKey("key"));
        
        if (result is RateLimitResult.Allowed allowed)
        {
            var results = await service.SearchAsync(allowed.Token, "query");
            Assert.NotNull(results);
        }
    }
}
```

## Why It's a Problem

1. **No compile-time enforcement**: Easy to forget rate limit checks
2. **Late checking**: Rate limits checked after expensive operations
3. **Inconsistent costs**: No way to charge different amounts
4. **Poor coordination**: Client retry logic not coordinated with server
5. **Manual tracking**: Must manually add rate limit headers

## Symptoms

- Rate limit middleware applied globally
- Rate limit checks scattered in controllers
- Missing rate limit checks on new endpoints
- Inconsistent rate limit responses
- No way to prioritize different operations

## Benefits

- **Compile-time safety**: Cannot call services without token
- **Early checking**: Rate limit checked before expensive work
- **Type-safe costs**: Different operations can cost different amounts
- **Coordinated retries**: Response includes when to retry
- **Composable**: Can combine multiple rate limiters (per-second, per-hour, quota)

## See Also

- [Capability Security](./capability-security.md) — tokens as proof
- [Authentication Context](./authentication-context.md) — identity tokens
- [Idempotency Keys](./idempotency-keys.md) — request deduplication
- [Strongly Typed IDs](./strongly-typed-ids.md) — client IDs
