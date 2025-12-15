# Type-Safe Circuit Breaker States

> Circuit breaker state machines with enums and runtime checks allow invalid transitions—use sealed class hierarchies to make illegal state transitions uncompilable.

## Problem

Circuit breaker implementations using enums or boolean flags to track state (Open, HalfOpen, Closed) allow invalid state transitions and require runtime validation. The compiler can't verify that state transitions follow the circuit breaker pattern.

## Example

### ❌ Before

```csharp
public enum CircuitState
{
    Closed,
    Open,
    HalfOpen
}

public class CircuitBreaker
{
    private CircuitState _state = CircuitState.Closed;
    private int _failureCount;
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation)
    {
        if (_state == CircuitState.Open)
            throw new CircuitBreakerOpenException();
        
        try
        {
            var result = await operation();
            if (_state == CircuitState.HalfOpen)
                _state = CircuitState.Closed;  // Runtime transition
            return result;
        }
        catch (Exception)
        {
            _failureCount++;
            if (_failureCount >= 3)
                _state = CircuitState.Open;  // Runtime transition
            throw;
        }
    }
}
```

### ✅ After

```csharp
public abstract record CircuitBreakerState
{
    private CircuitBreakerState() { }
    
    public sealed record Closed(int FailureCount) : CircuitBreakerState
    {
        public CircuitBreakerState RecordFailure(int threshold)
        {
            var newCount = FailureCount + 1;
            return newCount >= threshold
                ? new Open(DateTimeOffset.UtcNow)
                : new Closed(newCount);
        }
        
        public CircuitBreakerState RecordSuccess() => new Closed(0);
    }
    
    public sealed record Open(DateTimeOffset OpenedAt) : CircuitBreakerState
    {
        public Option<CircuitBreakerState> TryTransitionToHalfOpen(TimeSpan timeout)
        {
            var elapsed = DateTimeOffset.UtcNow - OpenedAt;
            return elapsed >= timeout
                ? Option<CircuitBreakerState>.Some(new HalfOpen())
                : Option<CircuitBreakerState>.None();
        }
    }
    
    public sealed record HalfOpen : CircuitBreakerState
    {
        public CircuitBreakerState RecordSuccess() => new Closed(0);
        public CircuitBreakerState RecordFailure() => new Open(DateTimeOffset.UtcNow);
    }
}

public sealed class TypeSafeCircuitBreaker
{
    private CircuitBreakerState _state = new CircuitBreakerState.Closed(0);
    private readonly int _failureThreshold;
    private readonly TimeSpan _openTimeout;
    
    public TypeSafeCircuitBreaker(int failureThreshold, TimeSpan openTimeout)
    {
        _failureThreshold = failureThreshold;
        _openTimeout = openTimeout;
    }
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation)
    {
        // Type-safe state handling
        var result = _state switch
        {
            CircuitBreakerState.Closed closed => await ExecuteClosedAsync(operation, closed),
            CircuitBreakerState.Open open => await ExecuteOpenAsync(operation, open),
            CircuitBreakerState.HalfOpen halfOpen => await ExecuteHalfOpenAsync(operation, halfOpen),
            _ => throw new InvalidOperationException("Unknown state")
        };
        
        return result;
    }
    
    private async Task<T> ExecuteClosedAsync<T>(
        Func<Task<T>> operation,
        CircuitBreakerState.Closed closed)
    {
        try
        {
            var result = await operation();
            _state = closed.RecordSuccess();
            return result;
        }
        catch (Exception)
        {
            _state = closed.RecordFailure(_failureThreshold);
            throw;
        }
    }
    
    private async Task<T> ExecuteOpenAsync<T>(
        Func<Task<T>> operation,
        CircuitBreakerState.Open open)
    {
        var halfOpenOption = open.TryTransitionToHalfOpen(_openTimeout);
        
        if (halfOpenOption.IsNone)
            throw new CircuitBreakerOpenException();
        
        _state = halfOpenOption.Value;
        return await ExecuteAsync(operation);
    }
    
    private async Task<T> ExecuteHalfOpenAsync<T>(
        Func<Task<T>> operation,
        CircuitBreakerState.HalfOpen halfOpen)
    {
        try
        {
            var result = await operation();
            _state = halfOpen.RecordSuccess();
            return result;
        }
        catch (Exception)
        {
            _state = halfOpen.RecordFailure();
            throw;
        }
    }
}
```

## See Also

- [Type-Level State Machines](./type-level-state-machines.md)
- [Type-Safe State Transitions](./type-safe-state-transitions.md)
- [Discriminated Unions](./discriminated-unions.md)
