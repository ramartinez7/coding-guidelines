# Cache Line Optimization

> Organize data structures to minimize cache misses and false sharing—align hot fields to cache boundaries for maximum throughput.

## Problem

Modern CPUs access memory in cache lines (typically 64 bytes). When multiple threads modify different fields in the same cache line, "false sharing" occurs—each write invalidates the cache line in other cores, causing expensive synchronization. Poorly organized data structures cause unnecessary cache misses, reducing throughput by 10-100x.

## Example

### ❌ Before (False Sharing)

```csharp
// ❌ Producer and consumer share cache line
public class Queue
{
    // Both on same cache line (64 bytes)
    private int head;      // Offset 0
    private int tail;      // Offset 4
    private object[] items;  // Offset 8

    // Problem: producer updates tail, consumer updates head
    // Every operation invalidates the cache line in the other core!
}

// Thread 1 (producer): writes to tail
// Thread 2 (consumer): writes to head
// Result: cache line bounces between cores
// Performance: 10x slower than it should be
```

### ✅ After (Cache Line Padding)

```csharp
// ✅ Separate hot fields into different cache lines
[StructLayout(LayoutKind.Explicit, Size = 192)]
public class Queue
{
    // Head in its own cache line (0-63)
    [FieldOffset(0)]
    private int head;

    [FieldOffset(64)]  // Start of cache line 2 (64-127)
    private int tail;

    [FieldOffset(128)]  // Start of cache line 3 (128-191)
    private object[] items;

    // Now producer and consumer operate on different cache lines
    // No false sharing!
}

// Performance: 10x faster!
```

## Cache Line Padding Pattern

```csharp
// ✅ Padding struct to prevent false sharing
[StructLayout(LayoutKind.Explicit, Size = 128)]
public struct PaddedLong
{
    [FieldOffset(64)]
    public long Value;

    // 64 bytes padding before
    // 64 bytes padding after
    // Total: 128 bytes (2 cache lines)
}

// Usage: separate counters for different threads
public class Counters
{
    private PaddedLong counter1;  // Cache line 1-2
    private PaddedLong counter2;  // Cache line 3-4
    private PaddedLong counter3;  // Cache line 5-6

    // No false sharing between counters
}
```

## Group Hot Fields Together

```csharp
// ❌ Fields scattered, poor locality
public class OrderBook
{
    private decimal bestBid;      // Hot
    private string exchange;      // Cold
    private decimal bestAsk;      // Hot
    private int orderId;          // Cold
    private int bidSize;          // Hot
    private DateTime lastUpdate;  // Cold
    private int askSize;          // Hot
}

// ✅ Group by access pattern
public class OrderBook
{
    // Hot fields (accessed together)
    private decimal bestBid;
    private decimal bestAsk;
    private int bidSize;
    private int askSize;

    // Padding to separate hot from cold
    private readonly long padding1;
    private readonly long padding2;
    private readonly long padding3;
    private readonly long padding4;
    private readonly long padding5;
    private readonly long padding6;
    private readonly long padding7;

    // Cold fields (accessed rarely)
    private string exchange;
    private int orderId;
    private DateTime lastUpdate;
}
```

## Struct Layout for Cache Efficiency

```csharp
// ❌ Poor layout: 40 bytes with padding
public struct TradeData
{
    public byte Type;       // 1 byte
                           // 7 bytes padding
    public long Timestamp;  // 8 bytes
    public decimal Price;   // 16 bytes
    public short Quantity;  // 2 bytes
                           // 6 bytes padding
}

// ✅ Optimized layout: 32 bytes, cache-line friendly
[StructLayout(LayoutKind.Explicit, Size = 32, Pack = 1)]
public struct TradeData
{
    [FieldOffset(0)]
    public long Timestamp;   // 8 bytes (align to 8)

    [FieldOffset(8)]
    public decimal Price;    // 16 bytes

    [FieldOffset(24)]
    public short Quantity;   // 2 bytes

    [FieldOffset(26)]
    public byte Type;        // 1 byte

    // 5 bytes padding to 32 (half cache line)
}
```

## Array of Structures vs Structure of Arrays

```csharp
// ❌ Array of Structures: poor cache utilization
public struct Quote
{
    public long Timestamp;   // 8 bytes
    public decimal Bid;      // 16 bytes
    public decimal Ask;      // 16 bytes
    public int Volume;       // 4 bytes
    // Total: 44 bytes per quote
}

public class QuoteArray
{
    private Quote[] quotes = new Quote[1000];

    // Problem: accessing only Bid prices loads entire Quote structs
    public decimal GetAverageBid()
    {
        decimal sum = 0;
        for (int i = 0; i < quotes.Length; i++)
        {
            sum += quotes[i].Bid;  // Loads 44 bytes to access 16
        }
        return sum / quotes.Length;
    }
}

// ✅ Structure of Arrays: better cache utilization
public class QuoteArraySOA
{
    private long[] timestamps = new long[1000];
    private decimal[] bids = new decimal[1000];
    private decimal[] asks = new decimal[1000];
    private int[] volumes = new int[1000];

    // Only loads Bid array (sequential, cache-friendly)
    public decimal GetAverageBid()
    {
        decimal sum = 0;
        for (int i = 0; i < bids.Length; i++)
        {
            sum += bids[i];  // Sequential access, perfect cache usage
        }
        return sum / bids.Length;
    }
}

// 3-4x faster for column-wise operations
```

## Aligning to Cache Line Boundaries

```csharp
// ✅ Align hot data structures to cache lines
[StructLayout(LayoutKind.Sequential)]
public unsafe struct CacheLineAligned
{
    public fixed byte padding[64];  // Force 64-byte alignment

    public static T* Allocate<T>() where T : unmanaged
    {
        // Allocate with 64-byte alignment
        var ptr = NativeMemory.AlignedAlloc(
            (nuint)sizeof(T),
            64);

        return (T*)ptr;
    }

    public static void Free<T>(T* ptr) where T : unmanaged
    {
        NativeMemory.AlignedFree(ptr);
    }
}

// Usage
var orderBook = CacheLineAligned.Allocate<OrderBook>();
// orderBook is now 64-byte aligned
```

## Prefetching

```csharp
public static class CacheOptimization
{
    // ✅ Prefetch data before use
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static void Prefetch(ref byte address)
    {
        // Hint to CPU to load cache line
        if (X86Base.IsSupported)
        {
            X86Base.Prefetch(ref address);
        }
    }

    // Usage in hot loop
    public static void ProcessOrders(Order[] orders)
    {
        for (int i = 0; i < orders.Length; i++)
        {
            // Prefetch next iteration's data
            if (i + 1 < orders.Length)
            {
                Prefetch(ref Unsafe.As<Order, byte>(ref orders[i + 1]));
            }

            ProcessOrder(orders[i]);
        }
    }
}
```

## Lock-Free Counter with Padding

```csharp
// ✅ Padded counter for different threads
[StructLayout(LayoutKind.Explicit, Size = 128)]
public struct PaddedCounter
{
    [FieldOffset(64)]
    private long value;

    public void Increment()
    {
        Interlocked.Increment(ref value);
    }

    public long Read()
    {
        return Interlocked.Read(ref value);
    }
}

public class ParallelCounters
{
    private readonly PaddedCounter[] counters;

    public ParallelCounters(int threadCount)
    {
        counters = new PaddedCounter[threadCount];
    }

    public void Increment(int threadId)
    {
        counters[threadId].Increment();  // No false sharing!
    }

    public long GetTotal()
    {
        long sum = 0;
        for (int i = 0; i < counters.Length; i++)
        {
            sum += counters[i].Read();
        }
        return sum;
    }
}
```

## Best Practices

### 1. Pad to Cache Line Size (64 bytes)

```csharp
// ✅ Use padding to separate hot fields
[StructLayout(LayoutKind.Explicit, Size = 192)]
public class ProducerConsumerQueue
{
    [FieldOffset(0)]
    private long head;  // Producer writes

    [FieldOffset(64)]   // Next cache line
    private long tail;  // Consumer writes

    [FieldOffset(128)]  // Another cache line
    private object[] buffer;
}
```

### 2. Measure Cache Misses

```csharp
// ✅ Use performance counters
var counters = new PerformanceCounterSampleBuilder()
    .AddCounter("Cache Misses/sec")
    .AddCounter("Cache Hit Ratio")
    .Build();

// Run workload and measure
var before = counters.Sample();
RunWorkload();
var after = counters.Sample();

var missRate = counters.Calculate("Cache Misses/sec", before, after);
```

### 3. Use Sequential Access Patterns

```csharp
// ❌ Random access: poor cache usage
for (int i = 0; i < 1000; i++)
{
    var index = random.Next(array.Length);
    Process(array[index]);  // Cache miss every time
}

// ✅ Sequential access: excellent cache usage
for (int i = 0; i < array.Length; i++)
{
    Process(array[i]);  // Prefetcher loads cache lines ahead
}
```

### 4. Keep Hot Data Small

```csharp
// ✅ Keep frequently accessed data under 64 bytes
public struct HotData
{
    public long Field1;    // 8 bytes
    public decimal Field2; // 16 bytes
    public int Field3;     // 4 bytes
    public int Field4;     // 4 bytes
    // Total: 32 bytes (fits in half cache line)
}
```

## Performance Impact

- **False sharing elimination**: 10-100x speedup
- **Cache-aligned access**: 2-5x speedup
- **SoA over AoS**: 3-4x speedup for columnar operations
- **Prefetching**: 10-30% improvement in sequential access

## When to Use

- Multi-threaded hot paths with shared data
- High-frequency data structure access
- Tight loops over arrays
- Lock-free algorithms

## When NOT to Use

- Single-threaded code (padding wastes memory)
- Data structures rarely accessed
- Memory footprint is critical concern

## Related Patterns

- [Lock-Free Data Structures](./lock-free-data-structures.md) — Contention-free concurrency
- [Struct Layout](../patterns/struct-layout.md) — Memory layout optimization
- [Hot Path Optimization](./hot-path-optimization.md) — Critical path performance
- [Thread Affinity](./thread-affinity.md) — CPU core pinning

## References

- "What Every Programmer Should Know About Memory" by Ulrich Drepper
- Intel® 64 and IA-32 Architectures Optimization Reference Manual
- "Avoiding and Identifying False Sharing Among Threads" - Intel
