# Circuit Breaker

> Prevent cascading failures by failing fast when a downstream service is unhealthy—stop calling a failing service until it recovers.

## Problem

When a downstream service fails or becomes slow, continuing to call it wastes resources and causes cascading failures. Clients wait for timeouts, consuming threads and memory. The failing service gets overwhelmed by requests it cannot handle, making recovery impossible.

## The Circuit Breaker Pattern

Like an electrical circuit breaker, this pattern "trips" when failures exceed a threshold, preventing further calls until the service recovers.

```
     Closed                      Open                       Half-Open
  (Normal Operation)         (Failing Fast)              (Testing Recovery)
        
  ┌──────────┐              ┌──────────┐                ┌──────────┐
  │  Client  │              │  Client  │                │  Client  │
  └────┬─────┘              └────┬─────┘                └────┬─────┘
       │                         │                           │
       │ Request                 │ Request                   │ Request
       │                         │                           │
       ▼                         ▼                           ▼
  ┌────────────┐            ┌────────────┐             ┌────────────┐
  │  Circuit   │            │  Circuit   │             │  Circuit   │
  │  Breaker   │            │  Breaker   │             │  Breaker   │
  │ (Closed)   │            │  (Open)    │             │ (Half-Open)│
  └────┬───────┘            └────┬───────┘             └────┬───────┘
       │                         │                           │
       │ Forward                 │ Fail Fast!                │ Try One
       │                         │                           │
       ▼                         ▼                           ▼
  ┌──────────┐              ┌──────────┐               ┌──────────┐
  │ Service  │              │ Return   │               │ Service  │
  │          │              │ Error    │               │          │
  └──────────┘              └──────────┘               └────┬─────┘
       │                                                     │
       │                                              Success? → Closed
  Success                                             Failure? → Open
```

## Example

### ❌ No Circuit Breaker

```csharp
public class PaymentService
{
    private readonly HttpClient _httpClient;
    
    public async Task<PaymentResult> ProcessPayment(Payment payment)
    {
        // If payment gateway is down, this will:
        // 1. Wait for timeout (e.g., 30 seconds)
        // 2. Consume a thread the entire time
        // 3. Repeat for every request
        // 4. Eventually exhaust thread pool
        
        try
        {
            var response = await _httpClient.PostAsync(
                "/charge",
                JsonContent.Create(payment));
            
            return await response.Content.ReadFromJsonAsync<PaymentResult>();
        }
        catch (HttpRequestException)
        {
            // Service is down, but we keep trying...
            throw;
        }
    }
}
```

### ✅ With Circuit Breaker

```csharp
/// <summary>
/// Circuit breaker states.
/// </summary>
public enum CircuitState
{
    Closed,    // Normal operation
    Open,      // Failing fast
    HalfOpen   // Testing recovery
}

/// <summary>
/// Circuit breaker implementation.
/// </summary>
public class CircuitBreaker
{
    private readonly int _failureThreshold;
    private readonly TimeSpan _openDuration;
    private readonly TimeSpan _timeout;
    
    private CircuitState _state = CircuitState.Closed;
    private int _failureCount = 0;
    private DateTimeOffset _lastFailureTime;
    private readonly SemaphoreSlim _lock = new(1, 1);
    
    public CircuitBreaker(
        int failureThreshold = 5,
        TimeSpan? openDuration = null,
        TimeSpan? timeout = null)
    {
        _failureThreshold = failureThreshold;
        _openDuration = openDuration ?? TimeSpan.FromSeconds(60);
        _timeout = timeout ?? TimeSpan.FromSeconds(10);
    }
    
    public async Task<Result<T, CircuitBreakerError>> ExecuteAsync<T>(
        Func<Task<T>> action)
    {
        await _lock.WaitAsync();
        
        try
        {
            // Check if circuit should transition from Open to Half-Open
            if (_state == CircuitState.Open)
            {
                if (DateTimeOffset.UtcNow - _lastFailureTime >= _openDuration)
                {
                    _state = CircuitState.HalfOpen;
                }
                else
                {
                    // Circuit is open, fail fast
                    return Result<T, CircuitBreakerError>.Failure(
                        new CircuitBreakerError.CircuitOpen(
                            $"Circuit is open. Retry after {_openDuration.TotalSeconds}s"));
                }
            }
        }
        finally
        {
            _lock.Release();
        }
        
        // Execute with timeout
        try
        {
            using var cts = new CancellationTokenSource(_timeout);
            var result = await action().WaitAsync(cts.Token);
            
            // Success—reset failure count
            await OnSuccess();
            
            return Result<T, CircuitBreakerError>.Success(result);
        }
        catch (OperationCanceledException)
        {
            // Timeout
            await OnFailure();
            return Result<T, CircuitBreakerError>.Failure(
                new CircuitBreakerError.Timeout("Operation timed out"));
        }
        catch (Exception ex)
        {
            // Failure
            await OnFailure();
            return Result<T, CircuitBreakerError>.Failure(
                new CircuitBreakerError.ServiceFailure(ex.Message));
        }
    }
    
    private async Task OnSuccess()
    {
        await _lock.WaitAsync();
        
        try
        {
            _failureCount = 0;
            
            if (_state == CircuitState.HalfOpen)
            {
                // Recovered! Close circuit
                _state = CircuitState.Closed;
            }
        }
        finally
        {
            _lock.Release();
        }
    }
    
    private async Task OnFailure()
    {
        await _lock.WaitAsync();
        
        try
        {
            _failureCount++;
            _lastFailureTime = DateTimeOffset.UtcNow;
            
            if (_state == CircuitState.HalfOpen)
            {
                // Still failing, open circuit again
                _state = CircuitState.Open;
            }
            else if (_failureCount >= _failureThreshold)
            {
                // Threshold exceeded, open circuit
                _state = CircuitState.Open;
            }
        }
        finally
        {
            _lock.Release();
        }
    }
    
    public CircuitState State => _state;
}

public abstract record CircuitBreakerError
{
    public sealed record CircuitOpen(string Message) : CircuitBreakerError;
    public sealed record Timeout(string Message) : CircuitBreakerError;
    public sealed record ServiceFailure(string Message) : CircuitBreakerError;
}

/// <summary>
/// Payment service with circuit breaker.
/// </summary>
public class ResilientPaymentService
{
    private readonly HttpClient _httpClient;
    private readonly CircuitBreaker _circuitBreaker;
    private readonly ILogger<ResilientPaymentService> _logger;
    
    public ResilientPaymentService(
        HttpClient httpClient,
        ILogger<ResilientPaymentService> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
        _circuitBreaker = new CircuitBreaker(
            failureThreshold: 5,
            openDuration: TimeSpan.FromSeconds(60),
            timeout: TimeSpan.FromSeconds(10));
    }
    
    public async Task<Result<PaymentResult, PaymentError>> ProcessPayment(Payment payment)
    {
        var result = await _circuitBreaker.ExecuteAsync(async () =>
        {
            var response = await _httpClient.PostAsync(
                "/charge",
                JsonContent.Create(payment));
            
            response.EnsureSuccessStatusCode();
            
            return await response.Content.ReadFromJsonAsync<PaymentResult>()
                ?? throw new InvalidOperationException("Empty response");
        });
        
        return result.Match<Result<PaymentResult, PaymentError>>(
            onSuccess: paymentResult => Result<PaymentResult, PaymentError>.Success(paymentResult),
            onFailure: error => error switch
            {
                CircuitBreakerError.CircuitOpen circuitOpen =>
                    Result<PaymentResult, PaymentError>.Failure(
                        new PaymentError.ServiceUnavailable(circuitOpen.Message)),
                
                CircuitBreakerError.Timeout timeout =>
                    Result<PaymentResult, PaymentError>.Failure(
                        new PaymentError.Timeout(timeout.Message)),
                
                CircuitBreakerError.ServiceFailure failure =>
                    Result<PaymentResult, PaymentError>.Failure(
                        new PaymentError.GatewayError(failure.Message)),
                
                _ => throw new InvalidOperationException($"Unknown error: {error}")
            });
    }
}
```

## Advanced: Per-Endpoint Circuit Breakers

Don't let one failing endpoint trip the circuit for all endpoints:

```csharp
public class CircuitBreakerFactory
{
    private readonly ConcurrentDictionary<string, CircuitBreaker> _breakers = new();
    
    public CircuitBreaker GetOrCreate(
        string key,
        int failureThreshold = 5,
        TimeSpan? openDuration = null,
        TimeSpan? timeout = null)
    {
        return _breakers.GetOrAdd(key, _ => new CircuitBreaker(
            failureThreshold,
            openDuration,
            timeout));
    }
}

public class MultiEndpointService
{
    private readonly HttpClient _httpClient;
    private readonly CircuitBreakerFactory _circuitBreakerFactory;
    
    public async Task<Result<User, ServiceError>> GetUser(UserId userId)
    {
        // Circuit breaker per operation
        var breaker = _circuitBreakerFactory.GetOrCreate("get-user");
        
        return await breaker.ExecuteAsync(async () =>
        {
            var response = await _httpClient.GetAsync($"/users/{userId}");
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<User>();
        });
    }
    
    public async Task<Result<List<Order>, ServiceError>> GetOrders(UserId userId)
    {
        // Different circuit breaker for different operation
        var breaker = _circuitBreakerFactory.GetOrCreate("get-orders");
        
        return await breaker.ExecuteAsync(async () =>
        {
            var response = await _httpClient.GetAsync($"/users/{userId}/orders");
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<List<Order>>();
        });
    }
}
```

## Fallback Strategies

When the circuit is open, provide degraded functionality:

```csharp
public class CachedPaymentService
{
    private readonly ResilientPaymentService _paymentService;
    private readonly ICache _cache;
    
    public async Task<Result<PaymentResult, PaymentError>> ProcessPayment(Payment payment)
    {
        var result = await _paymentService.ProcessPayment(payment);
        
        return await result.MatchAsync<Result<PaymentResult, PaymentError>>(
            onSuccess: async paymentResult =>
            {
                // Cache successful result
                await _cache.SetAsync($"payment-{payment.Id}", paymentResult);
                return Result<PaymentResult, PaymentError>.Success(paymentResult);
            },
            onFailure: async error => error switch
            {
                PaymentError.ServiceUnavailable =>
                    // Try cached data as fallback
                    await GetCachedResult(payment),
                
                _ => Result<PaymentResult, PaymentError>.Failure(error)
            });
    }
    
    private async Task<Result<PaymentResult, PaymentError>> GetCachedResult(Payment payment)
    {
        var cached = await _cache.GetAsync<PaymentResult>($"payment-{payment.Id}");
        
        return cached != null
            ? Result<PaymentResult, PaymentError>.Success(cached)
            : Result<PaymentResult, PaymentError>.Failure(
                new PaymentError.ServiceUnavailable("Service down and no cached data"));
    }
}
```

## Monitoring Circuit Breaker State

```csharp
public class ObservableCircuitBreaker : CircuitBreaker
{
    private readonly IMetrics _metrics;
    private readonly ILogger _logger;
    
    public event EventHandler<CircuitStateChangedEventArgs>? StateChanged;
    
    protected override async Task OnSuccess()
    {
        var previousState = State;
        await base.OnSuccess();
        
        if (previousState != State)
        {
            _logger.LogInformation(
                "Circuit breaker state changed: {Previous} → {Current}",
                previousState,
                State);
            
            StateChanged?.Invoke(this, new CircuitStateChangedEventArgs(
                previousState,
                State));
        }
        
        _metrics.Increment("circuit_breaker.success");
    }
    
    protected override async Task OnFailure()
    {
        var previousState = State;
        await base.OnFailure();
        
        if (previousState != State)
        {
            _logger.LogWarning(
                "Circuit breaker state changed: {Previous} → {Current}",
                previousState,
                State);
            
            StateChanged?.Invoke(this, new CircuitStateChangedEventArgs(
                previousState,
                State));
        }
        
        _metrics.Increment("circuit_breaker.failure");
    }
}

public class CircuitStateChangedEventArgs : EventArgs
{
    public CircuitState PreviousState { get; }
    public CircuitState CurrentState { get; }
    
    public CircuitStateChangedEventArgs(CircuitState previous, CircuitState current)
    {
        PreviousState = previous;
        CurrentState = current;
    }
}
```

## Why It's a Problem (Not Using Circuit Breakers)

1. **Cascading failures**: One slow service brings down everything
2. **Resource exhaustion**: Threads/connections consumed waiting for failures
3. **Slow recovery**: Failing service overwhelmed by requests it can't handle
4. **Poor user experience**: Long timeouts before errors

## Symptoms

- Thread pool exhaustion when downstream services fail
- Timeouts causing slow responses
- Services fail together (cascading failures)
- Recovery takes long time after issues resolved

## Benefits

- **Fast failure**: Immediate response when service is known to be down
- **Resource preservation**: Don't waste threads on known failures
- **Easier recovery**: Give failing service time to recover
- **Better user experience**: Fail fast instead of long timeouts

## See Also

- [Saga Pattern](./saga-pattern.md) — handling failures in distributed transactions
- [Event-Driven Architecture](./event-driven-architecture.md) — decoupling services to improve resilience
- [CAP Theorem](./cap-theorem.md) — understanding availability trade-offs
