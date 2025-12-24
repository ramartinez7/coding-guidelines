# SpinLock vs Mutex

> Choose the right synchronization primitive for your critical section—spinlocks for short waits, mutexes for longer operations.

## Problem

Synchronization primitives have different performance characteristics. Mutexes (locks) put threads to sleep when contended, adding ~2μs of context switch overhead. SpinLocks busy-wait without sleeping, wasting CPU but avoiding context switches. The right choice depends on critical section duration and contention patterns.

## Example

### ❌ Wrong Choice (Mutex for Short Critical Section)

```csharp
public class Counter
{
    private readonly object lockObj = new();
    private long value;

    public void Increment()
    {
        lock (lockObj)  // Mutex: context switch overhead
        {
            value++;  // 1 nanosecond operation
        }
        // Overhead: 2000x the actual work!
    }
}
```

### ✅ Right Choice (SpinLock for Short Critical Section)

```csharp
public class Counter
{
    private SpinLock spinLock = new(enableThreadOwnerTracking: false);
    private long value;

    public void Increment()
    {
        bool lockTaken = false;
        try
        {
            spinLock.Enter(ref lockTaken);
            value++;  // Fast, no context switch
        }
        finally
        {
            if (lockTaken)
                spinLock.Exit();
        }
    }
}
```

## Decision Matrix

```csharp
public static class SynchronizationChoice
{
    public static bool ShouldUseSpinLock(
        TimeSpan criticalSectionDuration,
        double contentionProbability)
    {
        // Rule of thumb:
        // - SpinLock: <1μs critical section, low contention
        // - Mutex: >1μs critical section or high contention

        if (criticalSectionDuration.TotalMicroseconds > 1)
            return false;  // Too long, use mutex

        if (contentionProbability > 0.5)
            return false;  // High contention, use mutex

        return true;  // Short and low contention: use spinlock
    }
}
```

## Mutex for I/O Operations

```csharp
// ✅ Mutex for longer operations
public class DatabaseConnection
{
    private readonly SemaphoreSlim semaphore = new(1, 1);

    public async Task<Result> QueryAsync(string sql)
    {
        await semaphore.WaitAsync();  // Async wait, thread can do other work
        try
        {
            return await ExecuteQueryAsync(sql);  // May take milliseconds
        }
        finally
        {
            semaphore.Release();
        }
    }
}
```

## SpinLock for Cache Updates

```csharp
// ✅ SpinLock for quick cache updates
public class FastCache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> cache = new();
    private SpinLock spinLock = new(enableThreadOwnerTracking: false);

    public TValue GetOrAdd(TKey key, Func<TKey, TValue> factory)
    {
        bool lockTaken = false;
        try
        {
            spinLock.Enter(ref lockTaken);

            if (cache.TryGetValue(key, out var value))
                return value;

            value = factory(key);
            cache[key] = value;
            return value;
        }
        finally
        {
            if (lockTaken)
                spinLock.Exit();
        }
    }
}
```

## Hybrid: Spin Then Wait

```csharp
// ✅ Best of both worlds
public class HybridLock
{
    private int locked;
    private readonly ManualResetEventSlim waitHandle = new(true);

    public void Enter()
    {
        // Try to acquire with spinning
        for (int i = 0; i < 100; i++)
        {
            if (Interlocked.CompareExchange(ref locked, 1, 0) == 0)
                return;

            Thread.SpinWait(10);
        }

        // Failed to acquire quickly, wait on event
        waitHandle.Wait();
        while (Interlocked.CompareExchange(ref locked, 1, 0) != 0)
        {
            waitHandle.Wait();
        }
    }

    public void Exit()
    {
        Interlocked.Exchange(ref locked, 0);
        waitHandle.Set();
    }
}
```

## Performance Comparison

```csharp
[MemoryDiagnoser]
public class LockBenchmarks
{
    private readonly object mutexLock = new();
    private SpinLock spinLock = new(false);
    private long counter;

    [Benchmark(Baseline = true)]
    public void Mutex()
    {
        lock (mutexLock)
        {
            counter++;
        }
    }

    [Benchmark]
    public void SpinLock()
    {
        bool taken = false;
        try
        {
            spinLock.Enter(ref taken);
            counter++;
        }
        finally
        {
            if (taken) spinLock.Exit();
        }
    }

    [Benchmark]
    public void InterlockedIncrement()
    {
        Interlocked.Increment(ref counter);
    }
}

// Results (typical):
// Mutex:               ~25ns (no contention), ~2000ns (with contention)
// SpinLock:            ~15ns (no contention), ~50ns (low contention)
// InterlockedIncrement: ~5ns (always fast)
```

## Best Practices

### 1. Measure Critical Section Duration

```csharp
// ✅ Profile to decide
var sw = Stopwatch.StartNew();
CriticalSection();
sw.Stop();

if (sw.Elapsed.TotalMicroseconds < 1)
{
    // Use SpinLock
}
```

### 2. Disable Thread Tracking in Production

```csharp
// ✅ Faster SpinLock without debugging overhead
private SpinLock spinLock = new(enableThreadOwnerTracking: false);
```

### 3. Consider Lock-Free Alternatives

```csharp
// ✅ Often better than any lock
private long counter;

public void Increment()
{
    Interlocked.Increment(ref counter);  // No lock needed
}
```

## When to Use

**SpinLock:**
- Critical section <1μs
- Low contention
- No I/O or blocking
- CPU cores available

**Mutex:**
- Critical section >1μs
- High contention possible
- I/O or blocking operations
- Shared CPU resources

**Lock-Free:**
- Simple atomic operations
- Highest performance needed
- Can tolerate retry loops

## Related Patterns

- [Lock-Free Data Structures](./lock-free-data-structures.md) — Avoid locks entirely
- [Thread Safety Patterns](../patterns/thread-safety-patterns.md) — General thread safety
- [Hot Path Optimization](./hot-path-optimization.md) — Critical path performance
