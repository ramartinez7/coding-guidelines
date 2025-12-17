# Thread Safety Patterns

> Write thread-safe code using immutability, proper synchronization, and concurrent collections to prevent race conditions and deadlocks.

## Problem

Shared mutable state accessed by multiple threads leads to race conditions, data corruption, and non-deterministic bugs that are hard to reproduce and debug.

## Example

### ❌ Before

```csharp
public class OrderCache
{
    private Dictionary<OrderId, Order> cache = new();

    // Race condition: not thread-safe!
    public void Add(Order order)
    {
        if (!cache.ContainsKey(order.Id))
        {
            cache[order.Id] = order;
        }
    }

    public Order? Get(OrderId id)
    {
        return cache.ContainsKey(id) ? cache[id] : null;
    }
}

public class Counter
{
    private int count = 0;

    // Race condition on read-modify-write
    public void Increment()
    {
        count++;  // Not atomic!
    }
}
```

### ✅ After

```csharp
public class OrderCache
{
    private readonly ConcurrentDictionary<OrderId, Order> cache = new();

    // Thread-safe with ConcurrentDictionary
    public void Add(Order order)
    {
        cache.TryAdd(order.Id, order);
    }

    public Option<Order> Get(OrderId id)
    {
        return cache.TryGetValue(id, out var order)
            ? Option.Some(order)
            : Option.None<Order>();
    }
}

public class Counter
{
    private int count = 0;

    // Thread-safe atomic increment
    public void Increment()
    {
        Interlocked.Increment(ref count);
    }

    public int GetCount()
    {
        return Interlocked.Read(ref count);
    }
}
```

## Best Practices

### 1. Prefer Immutability

```csharp
// ❌ Mutable shared state
public class UserSession
{
    public UserId UserId { get; set; }
    public DateTime LoginTime { get; set; }

    public void UpdateLoginTime()
    {
        LoginTime = DateTime.UtcNow;  // Race condition!
    }
}

// ✅ Immutable by default
public sealed record UserSession(
    UserId UserId,
    DateTime LoginTime)
{
    public UserSession UpdateLoginTime()
    {
        return this with { LoginTime = DateTime.UtcNow };
    }
}
```

### 2. Use Concurrent Collections

```csharp
// ❌ Dictionary with manual locking
public class Cache
{
    private readonly Dictionary<string, object> data = new();
    private readonly object lockObj = new();

    public void Add(string key, object value)
    {
        lock (lockObj)
        {
            data[key] = value;
        }
    }
}

// ✅ ConcurrentDictionary (lock-free for reads)
public class Cache
{
    private readonly ConcurrentDictionary<string, object> data = new();

    public void Add(string key, object value)
    {
        data[key] = value;
    }

    public Option<object> Get(string key)
    {
        return data.TryGetValue(key, out var value)
            ? Option.Some(value)
            : Option.None<object>();
    }
}
```

### 3. Use Interlocked for Atomic Operations

```csharp
// ❌ Non-atomic operations
private long totalRevenue;

public void AddRevenue(decimal amount)
{
    totalRevenue += (long)(amount * 100);  // Race condition!
}

// ✅ Atomic operations
private long totalRevenue;

public void AddRevenue(decimal amount)
{
    var cents = (long)(amount * 100);
    Interlocked.Add(ref totalRevenue, cents);
}
```

### 4. Use SemaphoreSlim for Async Coordination

```csharp
// ❌ lock with async (doesn't work!)
private readonly object lockObj = new();

public async Task ProcessAsync()
{
    lock (lockObj)  // Can't await inside lock!
    {
        await DoWorkAsync();
    }
}

// ✅ SemaphoreSlim for async
private readonly SemaphoreSlim semaphore = new(1, 1);

public async Task ProcessAsync()
{
    await semaphore.WaitAsync();
    try
    {
        await DoWorkAsync();
    }
    finally
    {
        semaphore.Release();
    }
}
```

### 5. Use ReaderWriterLockSlim for Read-Heavy Scenarios

```csharp
// ❌ Full lock even for reads
private readonly object lockObj = new();
private readonly Dictionary<string, int> data = new();

public int GetValue(string key)
{
    lock (lockObj)
    {
        return data[key];  // Blocks other readers!
    }
}

// ✅ ReaderWriterLockSlim allows concurrent reads
private readonly ReaderWriterLockSlim rwLock = new();
private readonly Dictionary<string, int> data = new();

public int GetValue(string key)
{
    rwLock.EnterReadLock();
    try
    {
        return data[key];
    }
    finally
    {
        rwLock.ExitReadLock();
    }
}

public void SetValue(string key, int value)
{
    rwLock.EnterWriteLock();
    try
    {
        data[key] = value;
    }
    finally
    {
        rwLock.ExitWriteLock();
    }
}
```

### 6. Use Lazy<T> for Thread-Safe Initialization

```csharp
// ❌ Double-checked locking (complex, error-prone)
private object lockObj = new();
private ExpensiveResource? resource;

public ExpensiveResource GetResource()
{
    if (resource == null)
    {
        lock (lockObj)
        {
            if (resource == null)
            {
                resource = new ExpensiveResource();
            }
        }
    }
    return resource;
}

// ✅ Lazy<T> handles thread safety
private readonly Lazy<ExpensiveResource> resource = 
    new(() => new ExpensiveResource());

public ExpensiveResource GetResource()
{
    return resource.Value;
}
```

### 7. Use ThreadLocal for Per-Thread State

```csharp
// ❌ Shared state with locking
private readonly object lockObj = new();
private readonly Random random = new();

public int GetRandomNumber()
{
    lock (lockObj)
    {
        return random.Next();  // Contention!
    }
}

// ✅ ThreadLocal eliminates contention
private readonly ThreadLocal<Random> random = 
    new(() => new Random());

public int GetRandomNumber()
{
    return random.Value!.Next();
}
```

### 8. Avoid Locking on Public Objects

```csharp
// ❌ Locking on this or public object
public class BadLocking
{
    public void Method()
    {
        lock (this)  // External code can lock too!
        {
            // Critical section
        }
    }
}

// ✅ Private lock object
public class GoodLocking
{
    private readonly object lockObj = new();

    public void Method()
    {
        lock (lockObj)
        {
            // Critical section
        }
    }
}
```

### 9. Use Channels for Producer-Consumer

```csharp
// ❌ Manual queue with locking
public class WorkQueue
{
    private readonly Queue<WorkItem> queue = new();
    private readonly object lockObj = new();

    public void Enqueue(WorkItem item)
    {
        lock (lockObj)
        {
            queue.Enqueue(item);
            Monitor.Pulse(lockObj);
        }
    }

    public WorkItem Dequeue()
    {
        lock (lockObj)
        {
            while (queue.Count == 0)
            {
                Monitor.Wait(lockObj);
            }
            return queue.Dequeue();
        }
    }
}

// ✅ Channel for producer-consumer
public class WorkQueue
{
    private readonly Channel<WorkItem> channel = 
        Channel.CreateUnbounded<WorkItem>();

    public async ValueTask EnqueueAsync(WorkItem item)
    {
        await channel.Writer.WriteAsync(item);
    }

    public async ValueTask<WorkItem> DequeueAsync(
        CancellationToken ct = default)
    {
        return await channel.Reader.ReadAsync(ct);
    }
}
```

### 10. Use Volatile for Simple Flags

```csharp
// ❌ Can be optimized away or reordered
private bool shouldStop = false;

public void Stop()
{
    shouldStop = true;
}

public void WorkLoop()
{
    while (!shouldStop)  // Compiler may cache this!
    {
        DoWork();
    }
}

// ✅ Volatile ensures visibility
private volatile bool shouldStop = false;

public void Stop()
{
    shouldStop = true;
}

public void WorkLoop()
{
    while (!shouldStop)
    {
        DoWork();
    }
}
```

### 11. Use ImmutableInterlocked for Immutable Updates

```csharp
// ❌ Lock for updating immutable collection
private readonly object lockObj = new();
private ImmutableList<string> items = ImmutableList<string>.Empty;

public void Add(string item)
{
    lock (lockObj)
    {
        items = items.Add(item);
    }
}

// ✅ ImmutableInterlocked (lock-free)
private ImmutableList<string> items = ImmutableList<string>.Empty;

public void Add(string item)
{
    ImmutableInterlocked.Update(ref items, list => list.Add(item));
}
```

### 12. Use CancellationToken for Cancellation

```csharp
// ❌ Polling volatile flag
private volatile bool shouldCancel = false;

public void LongRunningOperation()
{
    while (!shouldCancel)
    {
        ProcessItem();
    }
}

// ✅ CancellationToken
public void LongRunningOperation(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        ProcessItem();

        // Or throw if cancellation requested
        ct.ThrowIfCancellationRequested();
    }
}
```

## Symptoms

- Race conditions and data corruption
- Deadlocks that hang the application
- Non-deterministic test failures
- Performance bottlenecks from lock contention
- Inconsistent state between threads

## Benefits

- **Correct behavior** without race conditions
- **Better performance** with concurrent collections
- **Deadlock prevention** with proper locking patterns
- **Easier reasoning** with immutability
- **Scalability** with lock-free algorithms

## See Also

- [Immutable Collections](./immutable-collections.md) — Thread-safe by design
- [Value Semantics](./value-semantics.md) — Immutable value types
- [Async Patterns](./async-patterns.md) — Async coordination
