# Latency-Sensitive Design

> Design systems for consistent, predictable low latency—measure p99/p999 latencies and eliminate tail latency through careful design.

## Problem

Average latency is misleading. A system with 1μs average but 100ms p99 latency is unusable for trading. Tail latencies matter more than averages because they represent the worst-case experience. Unpredictable GC pauses, lock contention, and I/O blocking create latency spikes that kill trading systems.

## Example

### ❌ Before (Average-Focused)

```csharp
public class OrderProcessor
{
    public async Task<OrderResult> ProcessOrderAsync(Order order)
    {
        // Average case: 50μs
        // p99: 10ms (GC pause)
        // p999: 500ms (full GC)

        var result = await ValidateOrderAsync(order);  // Async I/O
        
        if (result.IsValid)
        {
            await _repository.SaveAsync(order);  // Database round-trip
            await _eventBus.PublishAsync(new OrderCreated(order));  // Message queue
        }

        return result;
    }
}

// Average: 50μs looks good!
// p99: 10ms unacceptable for HFT
// p999: 500ms catastrophic
```

### ✅ After (Latency-Focused)

```csharp
public class OrderProcessor
{
    private readonly RingBuffer<Order> buffer;
    private readonly OrderValidator validator;

    public OrderResult ProcessOrder(Order order)
    {
        // All operations synchronous, no allocations
        // p99: 2μs
        // p999: 5μs
        // p9999: 15μs (cache miss)

        var result = validator.Validate(order);  // In-memory, no I/O
        
        if (result.IsValid)
        {
            buffer.TryEnqueue(order);  // Lock-free, non-blocking
            // Background thread handles persistence asynchronously
        }

        return result;
    }
}

// Consistent low latency at all percentiles
```

## Measure All Percentiles

```csharp
public class LatencyTracker
{
    private readonly long[] samples;
    private int sampleIndex;
    private const int SampleCount = 10_000;

    public void RecordLatency(long latencyNanos)
    {
        samples[sampleIndex++ % SampleCount] = latencyNanos;
    }

    public LatencyStatistics GetStatistics()
    {
        var sorted = samples.OrderBy(x => x).ToArray();

        return new LatencyStatistics
        {
            Min = sorted[0],
            P50 = sorted[(int)(SampleCount * 0.50)],
            P90 = sorted[(int)(SampleCount * 0.90)],
            P99 = sorted[(int)(SampleCount * 0.99)],
            P999 = sorted[(int)(SampleCount * 0.999)],
            P9999 = sorted[(int)(SampleCount * 0.9999)],
            Max = sorted[SampleCount - 1],
            Average = sorted.Average()
        };
    }
}

public record LatencyStatistics
{
    public long Min { get; init; }
    public long P50 { get; init; }
    public long P90 { get; init; }
    public long P99 { get; init; }
    public long P999 { get; init; }
    public long P9999 { get; init; }
    public long Max { get; init; }
    public double Average { get; init; }
}

// Focus on p99 and p999, not average!
```

## Eliminate GC Pauses

```csharp
// ✅ Zero-allocation hot path
public class OrderMatcher
{
    private readonly ObjectPool<Order> orderPool;
    private readonly OrderBook book;

    public MatchResult Match(ReadOnlySpan<byte> orderData)
    {
        // Rent from pool (no allocation)
        var order = orderPool.Get();

        try
        {
            // Parse without allocating
            ParseOrder(orderData, order);

            // Match without allocating
            var result = book.TryMatch(order);

            return result;
        }
        finally
        {
            orderPool.Return(order);
        }
    }
}

// Result: No Gen0 collections, no GC pauses
```

## Synchronous-Only Critical Path

```csharp
// ❌ Async operations add latency
public async Task<Result> ProcessAsync()
{
    await Task.Delay(1);  // Thread pool scheduling
    return Result.Success;
}

// ✅ Synchronous for lowest latency
public Result Process()
{
    // Direct execution, no scheduling
    return Result.Success;
}

// ✅ Async for non-critical background work
public void ProcessOrder(Order order)
{
    var result = ValidateSync(order);  // Synchronous, critical path
    
    if (result.IsValid)
    {
        _ = PersistAsync(order);  // Fire-and-forget, non-critical
    }
}
```

## Lock-Free for Consistency

```csharp
// ❌ Locks cause contention
public class LockedQueue
{
    private readonly object lockObj = new();
    
    public void Enqueue(Order order)
    {
        lock (lockObj)  // Blocks, unpredictable latency
        {
            // ...
        }
    }
}

// ✅ Lock-free for consistent latency
public class LockFreeQueue
{
    private Node head;
    
    public void Enqueue(Order order)
    {
        // Compare-and-swap, bounded retries
        // Consistent latency
    }
}
```

## Pre-Allocate Resources

```csharp
// ❌ Allocate on demand
public class OrderProcessor
{
    public void Process(Order order)
    {
        var buffer = new byte[1024];  // Allocation!
        // ...
    }
}

// ✅ Pre-allocate at startup
public class OrderProcessor
{
    private readonly byte[] buffer = new byte[1024];
    private readonly ObjectPool<Order> pool;

    public OrderProcessor()
    {
        // Warm up pool
        for (int i = 0; i < 1000; i++)
        {
            var order = new Order();
            pool.Return(order);
        }
    }
}
```

## Eliminate Branch Mispredictions

```csharp
// ❌ Unpredictable branches
public decimal CalculatePrice(Order order)
{
    if (order.Type == OrderType.Market)
        return GetMarketPrice();
    else if (order.Type == OrderType.Limit)
        return order.LimitPrice;
    else if (order.Type == OrderType.Stop)
        return GetStopPrice();
    // ...
}

// ✅ Virtual dispatch (more predictable)
public abstract class Order
{
    public abstract decimal GetPrice();
}

public class MarketOrder : Order
{
    public override decimal GetPrice() => GetMarketPrice();
}
```

## Best Practices

### 1. Profile with High-Resolution Timers

```csharp
public class HighResolutionTimer
{
    private static readonly double TickFrequency = 
        (double)TimeSpan.TicksPerSecond / Stopwatch.Frequency;

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static long GetTimestampNanos()
    {
        return (long)(Stopwatch.GetTimestamp() * TickFrequency * 100);
    }
}
```

### 2. Use Busy-Wait for Ultra-Low Latency

```csharp
// ✅ Busy-wait for sub-microsecond latency
while (!IsDataAvailable())
{
    Thread.SpinWait(10);  // Spin, don't sleep
}
```

### 3. Disable GC in Critical Sections

```csharp
// ✅ Temporarily disable GC
GC.TryStartNoGCRegion(1024 * 1024);  // 1MB

try
{
    ProcessCriticalPath();  // No GC during this
}
finally
{
    GC.EndNoGCRegion();
}
```

### 4. Use Server GC

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

## When to Use

- High-frequency trading (HFT)
- Real-time gaming
- Industrial control systems
- Ultra-low latency requirements (<10μs p99)

## Related Patterns

- [Hot Path Optimization](./hot-path-optimization.md)
- [Predictable Performance](./predictable-performance.md)
- [Wait-Free Algorithms](./wait-free-algorithms.md)
- [Lock-Free Data Structures](./lock-free-data-structures.md)

## References

- "Systems Performance" by Brendan Gregg
- "The Every Computer Performance Book" by Bob Wescott
