# Type-Safe Retry Policies (Exponential Backoff Encoding)

> Retry logic with magic numbers and scattered configuration fails silently—encode retry policies in types to make retry behavior explicit and composable.

## Problem

Retry mechanisms implemented with loops, `Thread.Sleep`, and hardcoded delays are difficult to test, configure, and reason about. Retry behavior is scattered throughout the codebase with no type-level guarantees about backoff strategies or limits.

## Example

### ❌ Before

```csharp
public async Task<User> GetUserAsync(UserId id)
{
    int attempts = 0;
    while (attempts < 3)
    {
        try
        {
            return await _api.GetUserAsync(id);
        }
        catch (HttpRequestException)
        {
            attempts++;
            if (attempts >= 3)
                throw;
            
            // Magic numbers, linear backoff
            await Task.Delay(1000 * attempts);
        }
    }
    throw new Exception("Should never reach here");
}
```

### ✅ After

```csharp
public interface IRetryPolicy
{
    TimeSpan GetDelay(int attemptNumber);
    bool ShouldRetry(int attemptNumber);
}

public sealed class ExponentialBackoff : IRetryPolicy
{
    private readonly TimeSpan _initialDelay;
    private readonly int _maxAttempts;
    private readonly TimeSpan _maxDelay;
    
    public ExponentialBackoff(
        TimeSpan initialDelay,
        int maxAttempts,
        TimeSpan maxDelay)
    {
        _initialDelay = initialDelay;
        _maxAttempts = maxAttempts;
        _maxDelay = maxDelay;
    }
    
    public TimeSpan GetDelay(int attemptNumber)
    {
        var delay = _initialDelay * Math.Pow(2, attemptNumber - 1);
        return delay > _maxDelay ? _maxDelay : TimeSpan.FromMilliseconds(delay.TotalMilliseconds);
    }
    
    public bool ShouldRetry(int attemptNumber) => attemptNumber < _maxAttempts;
}

public static class RetryExtensions
{
    public static async Task<T> WithRetry<T>(
        this Task<T> operation,
        IRetryPolicy policy,
        Func<Exception, bool> shouldRetry)
    {
        int attempt = 0;
        while (true)
        {
            try
            {
                return await operation;
            }
            catch (Exception ex) when (shouldRetry(ex) && policy.ShouldRetry(attempt + 1))
            {
                attempt++;
                var delay = policy.GetDelay(attempt);
                await Task.Delay(delay);
            }
        }
    }
}

// Usage
var policy = new ExponentialBackoff(
    initialDelay: TimeSpan.FromSeconds(1),
    maxAttempts: 5,
    maxDelay: TimeSpan.FromSeconds(30));

var user = await _api.GetUserAsync(id)
    .WithRetry(policy, ex => ex is HttpRequestException);
```

## See Also

- [Type-Level Constraints](./type-level-constraints.md)
- [Phantom Types](./phantom-types.md)
