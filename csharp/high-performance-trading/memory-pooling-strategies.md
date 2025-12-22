# Memory Pooling Strategies

> Reuse objects instead of allocating new ones—use object pools, array pools, and memory pools to eliminate GC pressure in hot paths.

## Problem

Every allocation in .NET eventually triggers garbage collection. In high-throughput systems processing millions of operations per second, allocation pressure causes frequent GC pauses that add hundreds of microseconds of unpredictable latency. The GC, while efficient, cannot keep pace with extreme allocation rates in hot paths.

## Example

### ❌ Before (Allocating on Every Operation)

```csharp
public class MarketDataProcessor
{
    public void ProcessTick(ReadOnlySpan<byte> data)
    {
        // Allocation 1: Deserialize into new object
        var tick = ParseTick(data);

        // Allocation 2: Create buffer for processing
        var buffer = new byte[1024];

        // Allocation 3: Format message
        var message = $"Tick: {tick.Symbol} @ {tick.Price}";

        // Allocation 4: Create list for results
        var results = new List<string>();

        ProcessTickData(tick, buffer, results);
    }
}

// At 100k ticks/sec:
// - 400k+ allocations per second
// - GC Gen0 every ~10ms
// - p99 latency: 500μs (GC pause)
// - p999 latency: 5ms (Gen1/Gen2 collection)
```

### ✅ After (Using Object Pools)

```csharp
public class MarketDataProcessor
{
    private readonly ObjectPool<TickData> tickPool;
    private readonly ArrayPool<byte> bufferPool;
    private readonly ObjectPool<List<string>> listPool;

    public MarketDataProcessor()
    {
        this.tickPool = ObjectPool.Create<TickData>();
        this.bufferPool = ArrayPool<byte>.Shared;
        this.listPool = ObjectPool.Create(
            new ListPooledObjectPolicy<string>());
    }

    public void ProcessTick(ReadOnlySpan<byte> data)
    {
        // Rent from pools (zero allocation if available)
        var tick = tickPool.Get();
        var buffer = bufferPool.Rent(1024);
        var results = listPool.Get();

        try
        {
            // Reuse pooled objects
            ParseTickInto(data, tick);
            ProcessTickData(tick, buffer, results);

            // Use results...
        }
        finally
        {
            // Return to pools for reuse
            tick.Reset();
            tickPool.Return(tick);
            bufferPool.Return(buffer);
            results.Clear();
            listPool.Return(results);
        }
    }
}

// At 100k ticks/sec:
// - ~0 allocations in steady state
// - GC Gen0 rarely (only during warmup)
// - p99 latency: 5μs
// - p999 latency: 20μs
```

## ArrayPool Pattern

```csharp
public class BufferProcessor
{
    private readonly ArrayPool<byte> pool = ArrayPool<byte>.Shared;

    public void ProcessData(ReadOnlySpan<byte> input)
    {
        // ❌ Allocates new array every time
        // var buffer = new byte[4096];

        // ✅ Rent from pool
        var buffer = pool.Rent(4096);

        try
        {
            // Use buffer...
            input.CopyTo(buffer);
            ProcessBuffer(buffer.AsSpan(0, input.Length));
        }
        finally
        {
            // ✅ Return to pool
            pool.Return(buffer);
        }
    }

    // ✅ Alternative: use ArrayPool with using
    public void ProcessDataWithUsing(ReadOnlySpan<byte> input)
    {
        using var rental = MemoryPool<byte>.Shared.Rent(4096);
        var buffer = rental.Memory.Span;

        input.CopyTo(buffer);
        ProcessBuffer(buffer[..input.Length]);
    }
}
```

## Custom Object Pool

```csharp
public sealed class ObjectPool<T> where T : class
{
    private readonly ConcurrentBag<T> pool;
    private readonly Func<T> factory;
    private readonly Action<T>? reset;
    private readonly int maxSize;

    public ObjectPool(
        Func<T> factory,
        Action<T>? reset = null,
        int maxSize = 100)
    {
        this.pool = new ConcurrentBag<T>();
        this.factory = factory;
        this.reset = reset;
        this.maxSize = maxSize;
    }

    public T Get()
    {
        if (pool.TryTake(out var item))
        {
            return item;
        }

        // Pool empty, create new instance
        return factory();
    }

    public void Return(T item)
    {
        if (pool.Count < maxSize)
        {
            reset?.Invoke(item);
            pool.Add(item);
        }
        // Else discard (pool at capacity)
    }
}

// Usage
var orderPool = new ObjectPool<Order>(
    factory: () => new Order(),
    reset: order => order.Reset(),
    maxSize: 1000);

var order = orderPool.Get();
try
{
    // Use order...
}
finally
{
    orderPool.Return(order);
}
```

## Resettable Objects Pattern

```csharp
public interface IResettable
{
    void Reset();
}

public sealed class TickData : IResettable
{
    public string Symbol { get; set; } = "";
    public decimal Price { get; set; }
    public long Timestamp { get; set; }
    public int Volume { get; set; }

    public void Reset()
    {
        Symbol = "";
        Price = 0;
        Timestamp = 0;
        Volume = 0;
    }
}

public sealed class ResettableObjectPool<T> where T : class, IResettable, new()
{
    private readonly ConcurrentBag<T> pool = new();

    public T Get()
    {
        if (pool.TryTake(out var item))
        {
            return item;
        }

        return new T();
    }

    public void Return(T item)
    {
        item.Reset();
        pool.Add(item);
    }
}
```

## StringBuilder Pool

```csharp
public sealed class StringBuilderPool
{
    private static readonly ObjectPool<StringBuilder> Pool = ObjectPool.Create(
        new StringBuilderPooledObjectPolicy());

    public static StringBuilder Get()
    {
        return Pool.Get();
    }

    public static void Return(StringBuilder sb)
    {
        Pool.Return(sb);
    }

    public static string BuildString(Action<StringBuilder> build)
    {
        var sb = Get();
        try
        {
            build(sb);
            return sb.ToString();
        }
        finally
        {
            Return(sb);
        }
    }
}

// Usage
var message = StringBuilderPool.BuildString(sb =>
{
    sb.Append("Order: ");
    sb.Append(orderId);
    sb.Append(" @ ");
    sb.Append(price);
});
```

## Pool Monitoring and Metrics

```csharp
public sealed class MonitoredObjectPool<T> where T : class
{
    private readonly ObjectPool<T> pool;
    private long totalGets;
    private long totalReturns;
    private long totalAllocations;

    public long TotalGets => Interlocked.Read(ref totalGets);
    public long TotalReturns => Interlocked.Read(ref totalReturns);
    public long TotalAllocations => Interlocked.Read(ref totalAllocations);
    public long ActiveObjects => TotalGets - TotalReturns;

    public T Get()
    {
        Interlocked.Increment(ref totalGets);

        if (pool.TryGet(out var item))
        {
            return item;
        }

        Interlocked.Increment(ref totalAllocations);
        return CreateNew();
    }

    public void Return(T item)
    {
        Interlocked.Increment(ref totalReturns);
        pool.Return(item);
    }

    public PoolStatistics GetStatistics()
    {
        return new PoolStatistics
        {
            TotalGets = TotalGets,
            TotalReturns = TotalReturns,
            TotalAllocations = TotalAllocations,
            ActiveObjects = ActiveObjects,
            HitRate = TotalGets > 0
                ? (double)(TotalGets - TotalAllocations) / TotalGets
                : 0
        };
    }
}

public record PoolStatistics
{
    public long TotalGets { get; init; }
    public long TotalReturns { get; init; }
    public long TotalAllocations { get; init; }
    public long ActiveObjects { get; init; }
    public double HitRate { get; init; }  // % of Gets served from pool
}
```

## Per-Thread Pools

```csharp
// ✅ Eliminate contention with per-thread pools
public sealed class PerThreadObjectPool<T> where T : class, new()
{
    [ThreadStatic]
    private static Stack<T>? threadLocalPool;

    private const int MaxPerThreadSize = 100;

    public T Get()
    {
        threadLocalPool ??= new Stack<T>(MaxPerThreadSize);

        if (threadLocalPool.Count > 0)
        {
            return threadLocalPool.Pop();
        }

        return new T();
    }

    public void Return(T item)
    {
        if (threadLocalPool!.Count < MaxPerThreadSize)
        {
            threadLocalPool.Push(item);
        }
        // Discard if thread-local pool is full
    }
}

// Zero contention: each thread has its own pool
```

## Best Practices

### 1. Always Return to Pool

```csharp
// ✅ Use try-finally to ensure return
var buffer = pool.Rent(1024);
try
{
    // Use buffer...
}
finally
{
    pool.Return(buffer);
}

// ✅ Or use IDisposable wrapper
public readonly struct PooledArrayRental<T> : IDisposable
{
    private readonly ArrayPool<T> pool;
    private readonly T[] array;

    public PooledArrayRental(ArrayPool<T> pool, int minimumLength)
    {
        this.pool = pool;
        this.array = pool.Rent(minimumLength);
    }

    public Span<T> Span => array.AsSpan();

    public void Dispose()
    {
        pool.Return(array);
    }
}
```

### 2. Reset Pooled Objects

```csharp
// ✅ Clear state before returning to pool
public void Return(Order order)
{
    order.Id = default;
    order.Items.Clear();
    order.Status = OrderStatus.None;
    pool.Return(order);
}
```

### 3. Set Pool Size Appropriately

```csharp
// ❌ Pool too small: frequent allocations
var pool = new ObjectPool<T>(maxSize: 10);  // Only 10 objects

// ❌ Pool too large: memory waste
var pool = new ObjectPool<T>(maxSize: 1_000_000);  // Excessive

// ✅ Size based on concurrency + buffer
var expectedConcurrency = Environment.ProcessorCount * 2;
var pool = new ObjectPool<T>(maxSize: expectedConcurrency * 10);
```

### 4. Profile Pool Hit Rates

```csharp
// ✅ Monitor pool effectiveness
if (statistics.HitRate < 0.90)
{
    _logger.LogWarning(
        "Pool hit rate low: {HitRate:P}. Consider increasing pool size.",
        statistics.HitRate);
}
```

## Performance Characteristics

- **Get**: O(1) when pool has items, otherwise allocation cost
- **Return**: O(1)
- **Memory**: Constant memory usage (bounded by max size)
- **Contention**: ConcurrentBag has low contention, per-thread pools have none

## When to Use

- High-frequency allocations (>10k allocations/sec)
- GC pressure causing latency spikes
- Objects expensive to construct
- Need predictable memory usage

## When NOT to Use

- Low allocation rates
- Objects with complex lifecycle or cleanup
- Small objects with trivial construction cost
- Thread-local objects (use ThreadStatic instead)

## Related Patterns

- [Allocation Budget](../patterns/allocation-budget.md) — Zero-allocation techniques
- [Zero-Copy Techniques](./zero-copy-techniques.md) — Avoid copying data
- [Memory Safety (Span)](../patterns/memory-safety-span.md) — Stack allocation
- [Hot Path Optimization](./hot-path-optimization.md) — Optimize critical paths

## References

- Microsoft.Extensions.ObjectPool documentation
- System.Buffers.ArrayPool<T> documentation
- "Pro .NET Memory Management" by Konrad Kokosa
