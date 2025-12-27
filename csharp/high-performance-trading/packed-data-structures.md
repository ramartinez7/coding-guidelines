# Packed Data Structures

> Organize data for optimal cache utilization and memory density—use Structure-of-Arrays (SoA) layout and bit-packing for maximum throughput.

## Problem

Object-Oriented designs use Array-of-Structures (AoS) that scatter related data across memory, causing cache misses. When processing large datasets, the CPU spends more time waiting for memory than computing. Packed data structures group related fields together, maximize cache line utilization, and enable SIMD processing for 10-100x performance gains.

## Example

### ❌ Before (Array-of-Structures - Cache Inefficient)

```csharp
// ❌ AoS: Each object scattered in memory
public class Tick
{
    public long Timestamp;     // 8 bytes
    public decimal Price;      // 16 bytes
    public int Volume;         // 4 bytes
    public byte Side;          // 1 byte
    // Padding: ~7 bytes
    // Total: ~36 bytes per tick
}

public class TickProcessor
{
    private List<Tick> ticks = new();
    
    public decimal CalculateVWAP()
    {
        decimal sumPriceVolume = 0;
        int totalVolume = 0;
        
        // ❌ Poor cache utilization
        // Each iteration loads full Tick object (36 bytes)
        // But only uses Price (16 bytes) and Volume (4 bytes)
        // Wasted bandwidth: 16 bytes per tick!
        
        foreach (var tick in ticks)
        {
            sumPriceVolume += tick.Price * tick.Volume;
            totalVolume += tick.Volume;
        }
        
        return sumPriceVolume / totalVolume;
    }
    
    // ❌ Can't use SIMD efficiently
    // Data not contiguous
}
```

### ✅ After (Structure-of-Arrays - Cache Optimal)

```csharp
// ✅ SoA: Related data grouped together
public class TickDataSoA
{
    public long[] Timestamps;      // Contiguous array
    public decimal[] Prices;       // Contiguous array
    public int[] Volumes;          // Contiguous array
    public byte[] Sides;           // Contiguous array
    private int count;
    
    public TickDataSoA(int capacity)
    {
        Timestamps = new long[capacity];
        Prices = new decimal[capacity];
        Volumes = new int[capacity];
        Sides = new byte[capacity];
    }
    
    public void AddTick(long timestamp, decimal price, int volume, byte side)
    {
        Timestamps[count] = timestamp;
        Prices[count] = price;
        Volumes[count] = volume;
        Sides[count] = side;
        count++;
    }
    
    public decimal CalculateVWAP()
    {
        decimal sumPriceVolume = 0;
        int totalVolume = 0;
        
        // ✅ Excellent cache utilization
        // Prices and Volumes arrays accessed sequentially
        // CPU prefetcher loads next cache line ahead
        // All 64 bytes of cache line used!
        
        for (int i = 0; i < count; i++)
        {
            sumPriceVolume += Prices[i] * Volumes[i];
            totalVolume += Volumes[i];
        }
        
        return sumPriceVolume / totalVolume;
    }
    
    // ✅ Can use SIMD efficiently
    public unsafe decimal CalculateVWAPSIMD()
    {
        // Prices and Volumes are contiguous → perfect for SIMD
        // 5-10x faster than AoS version
        return 0m;  // Implementation using Vector<T>
    }
}
```

## Bit-Packed Structures

```csharp
// ✅ Pack multiple fields into single integer
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public readonly struct PackedTick
{
    // 64 bits total:
    // - 40 bits: price (fixed-point, 4 decimal places, range 0-109,951)
    // - 20 bits: volume (range 0-1,048,575)
    // - 3 bits: side + flags
    // - 1 bit: reserved
    
    private readonly ulong packed;
    
    private const ulong PriceMask = 0xFFFFFFFFFFul;      // 40 bits
    private const ulong VolumeMask = 0xFFFFFul;          // 20 bits
    private const ulong SideMask = 0x7ul;                // 3 bits
    
    private const int PriceShift = 0;
    private const int VolumeShift = 40;
    private const int SideShift = 60;
    
    public PackedTick(decimal price, int volume, byte side)
    {
        // Pack price (4 decimal places)
        ulong priceFixed = (ulong)(price * 10000m) & PriceMask;
        ulong volumePacked = ((ulong)volume & VolumeMask) << VolumeShift;
        ulong sidePacked = ((ulong)side & SideMask) << SideShift;
        
        packed = priceFixed | volumePacked | sidePacked;
    }
    
    public decimal Price =>
        (decimal)((packed >> PriceShift) & PriceMask) / 10000m;
    
    public int Volume =>
        (int)((packed >> VolumeShift) & VolumeMask);
    
    public byte Side =>
        (byte)((packed >> SideShift) & SideMask);
    
    // 8 bytes per tick vs 36 bytes in AoS
    // 4.5x memory savings!
}

// Usage:
public class PackedTickProcessor
{
    private PackedTick[] ticks = new PackedTick[10000];
    
    // ✅ 4.5x more ticks fit in cache
    // ✅ 4.5x less memory bandwidth
    // ✅ Can process 4.5x more ticks per second
}
```

## Hot-Cold Data Separation

```csharp
// ✅ Separate frequently-accessed (hot) from rarely-accessed (cold) data
public class Order
{
    // Hot data: accessed every tick
    public int OrderId;
    public decimal Price;
    public int Quantity;
    public byte State;
    
    // Cold data: accessed rarely
    public OrderDetails? Details;
}

public class OrderDetails
{
    public string ClientId;
    public string Account;
    public DateTime CreateTime;
    public string Trader;
    // etc...
}

// ✅ Hot path only loads Order (small)
// ✅ Cold data only loaded when needed
// ✅ Better cache utilization
```

## SIMD-Friendly Layouts

```csharp
public class SIMDFriendlyPriceData
{
    // ✅ Align to cache line (64 bytes)
    [StructLayout(LayoutKind.Sequential, Pack = 64)]
    public struct PriceBatch
    {
        public fixed float Prices[16];  // 64 bytes = 1 cache line
    }
    
    private PriceBatch[] batches;
    
    public SIMDFriendlyPriceData(int capacity)
    {
        int batchCount = (capacity + 15) / 16;
        batches = new PriceBatch[batchCount];
    }
    
    public unsafe void ProcessAllPrices()
    {
        foreach (ref var batch in batches.AsSpan())
        {
            fixed (float* ptr = batch.Prices)
            {
                // Load entire cache line
                Vector256<float> v1 = Avx.LoadVector256(ptr);
                Vector256<float> v2 = Avx.LoadVector256(ptr + 8);
                
                // Process 16 prices in 2 instructions
                v1 = Avx.Multiply(v1, Vector256.Create(1.01f));
                v2 = Avx.Multiply(v2, Vector256.Create(1.01f));
                
                Avx.Store(ptr, v1);
                Avx.Store(ptr + 8, v2);
            }
        }
    }
}
```

## Columnar Storage

```csharp
public class ColumnarOrderBook
{
    private const int MaxOrders = 10000;
    
    // ✅ Column-oriented storage
    private readonly decimal[] prices = new decimal[MaxOrders];
    private readonly int[] quantities = new int[MaxOrders];
    private readonly long[] timestamps = new long[MaxOrders];
    private readonly byte[] sides = new byte[MaxOrders];
    
    private int count;
    
    // ✅ Query by column (cache-friendly)
    public decimal GetAveragePrice()
    {
        decimal sum = 0;
        
        // Only loads price column
        for (int i = 0; i < count; i++)
        {
            sum += prices[i];
        }
        
        return sum / count;
    }
    
    // ✅ Filter by column
    public int CountBuyOrders()
    {
        int buyCount = 0;
        
        // Only loads side column
        for (int i = 0; i < count; i++)
        {
            if (sides[i] == 0)  // 0 = Buy
                buyCount++;
        }
        
        return buyCount;
    }
    
    // ✅ Multi-column aggregation
    public decimal GetBuyVolume()
    {
        decimal volume = 0;
        
        // Only loads relevant columns
        for (int i = 0; i < count; i++)
        {
            if (sides[i] == 0)
                volume += quantities[i];
        }
        
        return volume;
    }
}
```

## Compressed Timestamps

```csharp
public class CompressedTimestamps
{
    // ✅ Store delta from base (saves 4 bytes per timestamp)
    private readonly long baseTimestamp;
    private readonly int[] deltaMicroseconds;  // 4 bytes vs 8 bytes
    
    public CompressedTimestamps(int capacity, long baseTimestamp)
    {
        this.baseTimestamp = baseTimestamp;
        this.deltaMicroseconds = new int[capacity];
    }
    
    public void AddTimestamp(long timestamp, int index)
    {
        long delta = timestamp - baseTimestamp;
        deltaMicroseconds[index] = (int)(delta / 10);  // Store as microseconds
    }
    
    public long GetTimestamp(int index)
    {
        return baseTimestamp + (deltaMicroseconds[index] * 10L);
    }
    
    // 50% memory savings for timestamps
}
```

## Fixed-Point Decimals

```csharp
// ✅ Use fixed-point instead of decimal for speed
public readonly struct FixedPrice
{
    private readonly long value;  // 8 bytes vs 16 bytes for decimal
    private const int Scale = 10000;  // 4 decimal places
    
    public FixedPrice(decimal price)
    {
        value = (long)(price * Scale);
    }
    
    public decimal ToDecimal() => (decimal)value / Scale;
    
    // ✅ Much faster arithmetic (integer ops vs decimal ops)
    public static FixedPrice operator +(FixedPrice a, FixedPrice b)
        => new() { value = a.value + b.value };
    
    public static FixedPrice operator *(FixedPrice a, int multiplier)
        => new() { value = a.value * multiplier };
    
    // 50% memory savings, 10x faster arithmetic
}
```

## Parallel Array Pattern

```csharp
public class ParallelArrays
{
    // ✅ Separate arrays for different access patterns
    
    // Read-mostly data
    private readonly decimal[] staticPrices = new decimal[10000];
    private readonly string[] symbols = new string[10000];
    
    // Write-heavy data
    private readonly int[] volumes = new int[10000];
    private readonly long[] lastUpdateTimes = new long[10000];
    
    // ✅ Hot data (frequent access)
    private readonly byte[] activeFlags = new byte[10000];
    
    // ✅ Cold data (rare access)
    private readonly string[] descriptions = new string[10000];
    
    public void UpdateVolume(int index, int volume)
    {
        // Only touches write-heavy array
        volumes[index] = volume;
        lastUpdateTimes[index] = DateTime.UtcNow.Ticks;
    }
    
    public bool IsActive(int index)
    {
        // Only touches hot data
        return activeFlags[index] != 0;
    }
}
```

## Bitmap Indices

```csharp
public class BitmapIndex
{
    // ✅ Use bitmaps for boolean queries
    private ulong[] activeMask = new ulong[16];  // 1024 bits
    private ulong[] buyMask = new ulong[16];
    
    public void SetActive(int index, bool active)
    {
        int wordIndex = index / 64;
        int bitIndex = index % 64;
        ulong mask = 1ul << bitIndex;
        
        if (active)
            activeMask[wordIndex] |= mask;
        else
            activeMask[wordIndex] &= ~mask;
    }
    
    public int CountActiveBuys()
    {
        int count = 0;
        
        // ✅ Extremely fast bitwise AND + popcount
        for (int i = 0; i < activeMask.Length; i++)
        {
            ulong combined = activeMask[i] & buyMask[i];
            count += BitOperations.PopCount(combined);
        }
        
        return count;
    }
    
    // 64x memory savings vs byte array
    // 100x faster queries
}
```

## Best Practices

### 1. Profile Before Packing

```csharp
// ✅ Measure memory bandwidth and cache misses
// Only pack if bottleneck is memory-bound
```

### 2. Align to Cache Lines

```csharp
// ✅ Align hot data to 64-byte boundaries
[StructLayout(LayoutKind.Sequential, Pack = 64)]
public struct CacheAligned
{
    public fixed byte Data[64];
}
```

### 3. Group by Access Pattern

```csharp
// ✅ Group fields accessed together
// Separate hot from cold data
```

### 4. Use Power-of-2 Sizes

```csharp
// ✅ Makes indexing faster (no division)
int capacity = 1024;  // 2^10
int index = hash & (capacity - 1);  // Fast modulo
```

## Performance Characteristics

- **Cache efficiency**: 2-10x improvement
- **Memory bandwidth**: 2-5x reduction
- **SIMD throughput**: 4-16x improvement
- **Memory footprint**: 2-10x reduction

## When to Use

- Processing large arrays/collections
- Memory-bandwidth limited workloads
- SIMD-heavy computations
- Cache-sensitive algorithms
- Embedded systems (memory constrained)

## When NOT to Use

- Small datasets (<1000 elements)
- Random access patterns
- Complex object relationships
- Maintainability priority over performance

## Related Patterns

- [SIMD Vectorization](./simd-vectorization.md) — Vector operations
- [Cache Line Optimization](./cache-line-optimization.md) — Cache friendly
- [Bit Manipulation](./bit-manipulation-techniques.md) — Bit packing
- [Struct Layout](../patterns/struct-layout.md) — Memory layout

## References

- "Data-Oriented Design" by Richard Fabian
- "Structure of Arrays vs Array of Structures" - Intel
- "Cache-Oblivious Algorithms" - MIT
- "Columnar Storage" - ClickHouse documentation
