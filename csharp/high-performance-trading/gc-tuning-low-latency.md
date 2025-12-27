# GC Tuning for Low-Latency Systems

> Minimize garbage collection pauses for predictable latency—configure GC for trading systems where millisecond pauses are unacceptable.

## Problem

The .NET garbage collector stops application threads to collect garbage. In low-latency trading systems, even a 10ms GC pause can cause missed trades, stale quotes, or regulatory violations. Default GC settings prioritize throughput over latency, leading to unpredictable pauses that destroy p99 latency guarantees.

## Example

### ❌ Before (Default GC, Unpredictable Latency)

```csharp
public class HighAllocationTrading
{
    public void ProcessMarketData(byte[] rawData)
    {
        // ❌ Allocates on every call
        var message = Encoding.UTF8.GetString(rawData);
        var parts = message.Split('|');
        var order = new Order
        {
            Symbol = parts[0],
            Price = decimal.Parse(parts[1]),
            Quantity = int.Parse(parts[2])
        };
        
        ProcessOrder(order);
        
        // Gen0: 10MB allocated
        // GC pause every few seconds: 5-50ms
        // p99 latency: 50ms+ (unacceptable!)
    }
    
    private void ProcessOrder(Order order) { }
}

public class Order
{
    public string Symbol { get; set; } = "";
    public decimal Price { get; set; }
    public int Quantity { get; set; }
}
```

### ✅ After (GC-Optimized, Predictable Latency)

```csharp
public class LowAllocationTrading
{
    private readonly ArrayPool<byte> bufferPool = ArrayPool<byte>.Shared;
    private readonly OrderPool orderPool = new();
    
    public void ProcessMarketData(ReadOnlySpan<byte> rawData)
    {
        // ✅ Zero allocations using Span
        Span<Range> parts = stackalloc Range[3];
        var chars = stackalloc char[rawData.Length];
        Encoding.UTF8.GetChars(rawData, chars);
        
        var charSpan = new ReadOnlySpan<char>(chars, rawData.Length);
        Split(charSpan, '|', parts);
        
        // ✅ Rent from pool (no allocation)
        var order = orderPool.Rent();
        
        order.Symbol = new string(charSpan[parts[0]]);
        order.Price = decimal.Parse(charSpan[parts[1]]);
        order.Quantity = int.Parse(charSpan[parts[2]]);
        
        ProcessOrder(order);
        
        // ✅ Return to pool
        orderPool.Return(order);
        
        // Gen0 allocations: near zero
        // GC pauses: rare, <1ms
        // p99 latency: <1ms
    }
    
    private void Split(ReadOnlySpan<char> input, char separator, Span<Range> ranges)
    {
        int rangeCount = 0;
        int start = 0;
        
        for (int i = 0; i < input.Length && rangeCount < ranges.Length; i++)
        {
            if (input[i] == separator)
            {
                ranges[rangeCount++] = new Range(start, i);
                start = i + 1;
            }
        }
        
        if (start < input.Length && rangeCount < ranges.Length)
        {
            ranges[rangeCount] = new Range(start, input.Length);
        }
    }
    
    private void ProcessOrder(Order order) { }
}
```

## GC Configuration Options

```csharp
public static class GCConfiguration
{
    // ✅ Enable Server GC (multi-threaded, higher throughput)
    // In .csproj or runtimeconfig.json:
    // <ServerGarbageCollection>true</ServerGarbageCollection>
    
    // ✅ Enable Concurrent GC (background collection)
    // <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
    
    // ✅ Set GC latency mode
    public static void ConfigureLowLatency()
    {
        // Sustained Low Latency: minimize pause times
        GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;
        
        // Or even more aggressive (but higher memory):
        // GCSettings.LatencyMode = GCLatencyMode.LowLatency;
    }
    
    // ✅ NoGCRegion for critical sections
    public static void CriticalSection()
    {
        // Disable GC for critical operation
        if (GC.TryStartNoGCRegion(1024 * 1024))  // 1MB budget
        {
            try
            {
                // Critical code here
                ProcessLatencySensitiveOperation();
            }
            finally
            {
                GC.EndNoGCRegion();
            }
        }
    }
    
    // ✅ Large object heap compaction
    public static void CompactLOH()
    {
        // Compact LOH to reduce fragmentation
        GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
        GC.Collect();
    }
    
    // ✅ Gen2 region size (server GC)
    public static void ConfigureGen2RegionSize()
    {
        // Set via environment variable before startup:
        // COMPlus_GCgen2regionSize=0x4000000  (64MB)
        // Larger regions = fewer Gen2 collections
    }
    
    private static void ProcessLatencySensitiveOperation() { }
}
```

## Allocation Budget Pattern

```csharp
public class AllocationBudget
{
    private long startMemory;
    private readonly long maxAllocation;
    
    public AllocationBudget(long maxAllocationBytes)
    {
        this.maxAllocation = maxAllocationBytes;
        this.startMemory = GC.GetTotalMemory(false);
    }
    
    public void CheckBudget()
    {
        long currentMemory = GC.GetTotalMemory(false);
        long allocated = currentMemory - startMemory;
        
        if (allocated > maxAllocation)
        {
            throw new InvalidOperationException(
                $"Allocation budget exceeded: {allocated} > {maxAllocation}");
        }
    }
    
    public void Reset()
    {
        startMemory = GC.GetTotalMemory(false);
    }
}

// Usage:
public class TradingEngine
{
    private readonly AllocationBudget budget = new(1024 * 1024);  // 1MB budget
    
    public void ProcessTick()
    {
        budget.Reset();
        
        // Process market data
        ProcessMarketData();
        
        // Verify we stayed within budget
        budget.CheckBudget();
    }
    
    private void ProcessMarketData() { }
}
```

## Object Pooling

```csharp
public class ObjectPool<T> where T : class, new()
{
    private readonly ConcurrentBag<T> pool = new();
    private readonly int maxSize;
    
    public ObjectPool(int maxSize = 1000)
    {
        this.maxSize = maxSize;
    }
    
    public T Rent()
    {
        if (pool.TryTake(out var item))
        {
            return item;
        }
        
        return new T();  // Allocate if pool empty
    }
    
    public void Return(T item)
    {
        if (pool.Count < maxSize)
        {
            pool.Add(item);
        }
        // Else let GC collect it
    }
    
    public void Preallocate(int count)
    {
        for (int i = 0; i < count; i++)
        {
            pool.Add(new T());
        }
    }
}

// Usage:
public class OrderPool
{
    private readonly ObjectPool<Order> pool = new(10000);
    
    public OrderPool()
    {
        // Preallocate during startup
        pool.Preallocate(1000);
    }
    
    public Order Rent() => pool.Rent();
    
    public void Return(Order order)
    {
        // Reset state
        order.Symbol = "";
        order.Price = 0;
        order.Quantity = 0;
        
        pool.Return(order);
    }
}
```

## Gen0 Budget Monitoring

```csharp
public class Gen0Monitor
{
    private int lastGen0Count;
    
    public Gen0Monitor()
    {
        this.lastGen0Count = GC.CollectionCount(0);
    }
    
    public int GetGen0CollectionsSinceLastCheck()
    {
        int current = GC.CollectionCount(0);
        int count = current - lastGen0Count;
        lastGen0Count = current;
        return count;
    }
    
    public void WarnIfGen0Collection()
    {
        int gen0Collections = GetGen0CollectionsSinceLastCheck();
        
        if (gen0Collections > 0)
        {
            Console.WriteLine($"WARNING: {gen0Collections} Gen0 GC(s) occurred");
            
            // Log stack trace to find allocation source
            Console.WriteLine(Environment.StackTrace);
        }
    }
}

// Usage in hot path:
public class TradingLoop
{
    private readonly Gen0Monitor monitor = new();
    
    public void ProcessTick()
    {
        // Process data
        ProcessMarketData();
        
        // Check if GC occurred
        monitor.WarnIfGen0Collection();
    }
    
    private void ProcessMarketData() { }
}
```

## Pinned Object Heap (.NET 5+)

```csharp
public class PinnedObjectHeap
{
    // ✅ Allocate on Pinned Object Heap (POH)
    public static byte[] AllocatePinned(int size)
    {
        // POH: never moved by GC, no pinning cost
        return GC.AllocateArray<byte>(size, pinned: true);
    }
    
    // Usage: long-lived buffers that need pinning
    private readonly byte[] receiveBuffer;
    
    public PinnedObjectHeap()
    {
        // Allocate 1MB buffer on POH
        receiveBuffer = GC.AllocateArray<byte>(1024 * 1024, pinned: true);
    }
    
    public unsafe void ProcessData(int length)
    {
        fixed (byte* ptr = receiveBuffer)  // No-op: already pinned!
        {
            // Use pointer
            ProcessPointer(ptr, length);
        }
    }
    
    private unsafe void ProcessPointer(byte* ptr, int length) { }
}
```

## Struct-Based Design

```csharp
// ✅ Value types avoid heap allocations
public readonly record struct TickData(
    long Timestamp,
    decimal Price,
    int Volume,
    byte Side  // 0 = Buy, 1 = Sell
);

public class TickProcessor
{
    private TickData[] ringBuffer = new TickData[10000];  // Single allocation
    private int writeIndex;
    
    public void AddTick(long timestamp, decimal price, int volume, byte side)
    {
        // ✅ No allocation: struct copied into array
        ringBuffer[writeIndex] = new TickData(timestamp, price, volume, side);
        writeIndex = (writeIndex + 1) % ringBuffer.Length;
    }
    
    public TickData GetTick(int index)
    {
        return ringBuffer[index];  // Copy, not reference
    }
}
```

## Startup Allocation Strategy

```csharp
public class TradingSystem
{
    public void Initialize()
    {
        // ✅ Allocate everything at startup
        PreallocateBuffers();
        PreallocatePools();
        WarmupJIT();
        
        // Force Gen2 collection to promote everything
        GC.Collect(2, GCCollectionMode.Forced, blocking: true);
        GC.WaitForPendingFinalizers();
        GC.Collect(2, GCCollectionMode.Forced, blocking: true);
        
        // Now in Gen2, won't be collected during trading
        
        // Switch to low-latency mode
        GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;
    }
    
    private void PreallocateBuffers()
    {
        // Allocate all buffers up front
    }
    
    private void PreallocatePools()
    {
        // Fill object pools
    }
    
    private void WarmupJIT()
    {
        // Exercise all hot paths to trigger JIT compilation
    }
}
```

## Measuring GC Impact

```csharp
public class GCMetrics
{
    public static void ReportGCStats()
    {
        Console.WriteLine("=== GC Statistics ===");
        Console.WriteLine($"Gen0 collections: {GC.CollectionCount(0)}");
        Console.WriteLine($"Gen1 collections: {GC.CollectionCount(1)}");
        Console.WriteLine($"Gen2 collections: {GC.CollectionCount(2)}");
        Console.WriteLine($"Total memory: {GC.GetTotalMemory(false) / 1024.0 / 1024.0:F2} MB");
        Console.WriteLine($"GC mode: {(GCSettings.IsServerGC ? "Server" : "Workstation")}");
        Console.WriteLine($"Latency mode: {GCSettings.LatencyMode}");
        
        // GC info (.NET 5+)
        var gcInfo = GC.GetGCMemoryInfo();
        Console.WriteLine($"Heap size: {gcInfo.HeapSizeBytes / 1024.0 / 1024.0:F2} MB");
        Console.WriteLine($"Fragmented: {gcInfo.FragmentedBytes / 1024.0 / 1024.0:F2} MB");
        Console.WriteLine($"Generation: {gcInfo.Generation}");
        Console.WriteLine($"Compacting: {gcInfo.Compacted}");
    }
    
    public static void MeasureAllocationRate(Action action, int iterations)
    {
        long before = GC.GetTotalAllocatedBytes(precise: true);
        
        for (int i = 0; i < iterations; i++)
        {
            action();
        }
        
        long after = GC.GetTotalAllocatedBytes(precise: true);
        long allocated = after - before;
        
        Console.WriteLine($"Allocated: {allocated} bytes over {iterations} iterations");
        Console.WriteLine($"Per iteration: {allocated / iterations} bytes");
    }
}
```

## Best Practices

### 1. Use Span<T> to Avoid Allocations

```csharp
// ✅ Span avoids string allocations
public void Parse(ReadOnlySpan<char> input)
{
    // No allocations
}

// ❌ String creates allocations
public void Parse(string input)
{
    var parts = input.Split('|');  // Allocates array + strings
}
```

### 2. Pool Long-Lived Objects

```csharp
// ✅ Pool objects that live > 1 tick
ObjectPool<Order> orderPool = new(10000);
```

### 3. Use Structs for Data

```csharp
// ✅ Struct = stack or inline, no GC pressure
public readonly record struct Quote(decimal Bid, decimal Ask);
```

### 4. Preallocate at Startup

```csharp
// ✅ Allocate during initialization
byte[][] buffers = new byte[1000][];
for (int i = 0; i < buffers.Length; i++)
    buffers[i] = new byte[4096];
```

### 5. Monitor GC in Production

```csharp
// ✅ Alert on unexpected collections
if (GC.CollectionCount(0) > expectedCount)
    AlertOps();
```

## Performance Characteristics

- **Default GC**: 5-50ms pauses
- **Sustained Low Latency**: 1-5ms pauses
- **Zero allocations**: No GC pauses
- **Server GC**: Higher throughput, larger pauses
- **Workstation GC**: Lower latency, lower throughput

## When to Use

- High-frequency trading systems
- Real-time gaming engines
- Ultra-low latency messaging
- Signal processing
- Any system with p99 < 10ms requirement

## When NOT to Use

- Standard web applications
- Batch processing
- Long-running operations (minutes+)
- Memory-intensive workloads

## Related Patterns

- [Memory Pooling Strategies](./memory-pooling-strategies.md) — Object pools
- [Zero-Copy Techniques](./zero-copy-techniques.md) — Span usage
- [Allocation Budget](../patterns/allocation-budget.md) — Track allocations
- [Struct Value Types](./struct-value-types-performance.md) — Value semantics

## References

- ".NET Garbage Collection" - Microsoft Docs
- "Pro .NET Memory Management" by Konrad Kokosa
- "Writing High-Performance .NET Code" by Ben Watson
- "GC Tuning Guide" - .NET Performance
