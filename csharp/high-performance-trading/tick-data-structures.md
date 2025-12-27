# Tick Data Structures

> Design efficient data structures for tick-by-tick market data—optimize for sequential writes and minimal memory footprint.

## Problem

Trading systems store billions of ticks for analysis, backtesting, and compliance. Traditional row-based storage wastes memory and causes cache misses. Efficient tick structures use columnar storage, compression, and memory-mapped files to handle massive datasets with minimal overhead.

## Example

### ❌ Before (Row-Based, Wasteful)

```csharp
public class Tick
{
    public string Symbol { get; set; } = "";  // 8 bytes pointer + string heap
    public DateTime Timestamp { get; set; }   // 8 bytes
    public decimal Price { get; set; }        // 16 bytes
    public int Volume { get; set; }           // 4 bytes
    public byte Flags { get; set; }           // 1 byte
    // Total: ~40 bytes + heap allocations
}

public class TickStorage
{
    private List<Tick> ticks = new();  // Each tick is separate object

    public void Add(Tick tick)
    {
        ticks.Add(tick);  // Object allocation + list resize
    }
}

// 1 million ticks = ~40MB + GC pressure
```

### ✅ After (Columnar, Compact)

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct CompactTick
{
    public long TimestampTicks;  // 8 bytes
    public int PriceTicks;       // 4 bytes (scaled integer)
    public int Volume;           // 4 bytes
    public byte Flags;           // 1 byte
    // Total: 17 bytes, no padding
}

public class ColumnarTickStorage
{
    private readonly long[] timestamps;
    private readonly int[] prices;
    private readonly int[] volumes;
    private readonly byte[] flags;
    private int count;

    public ColumnarTickStorage(int capacity)
    {
        timestamps = new long[capacity];
        prices = new int[capacity];
        volumes = new int[capacity];
        flags = new byte[capacity];
    }

    public void Add(long timestamp, decimal price, int volume, byte flags)
    {
        timestamps[count] = timestamp;
        prices[count] = (int)(price * 10000);  // Scale to int
        volumes[count] = volume;
        this.flags[count] = flags;
        count++;
    }

    public ReadOnlySpan<long> GetTimestamps() => timestamps.AsSpan(0, count);
    public ReadOnlySpan<int> GetPrices() => prices.AsSpan(0, count);
}

// 1 million ticks = ~17MB, no GC, sequential access
```

## Memory-Mapped Tick Storage

```csharp
public class MemoryMappedTickStorage : IDisposable
{
    private readonly MemoryMappedFile mmf;
    private readonly MemoryMappedViewAccessor accessor;
    private long position;

    public MemoryMappedTickStorage(string filename, long capacity)
    {
        mmf = MemoryMappedFile.CreateFromFile(
            filename,
            FileMode.Create,
            null,
            capacity * 17);  // 17 bytes per tick

        accessor = mmf.CreateViewAccessor();
    }

    public void WriteTick(long timestamp, decimal price, int volume, byte flags)
    {
        accessor.Write(position, timestamp);
        accessor.Write(position + 8, (int)(price * 10000));
        accessor.Write(position + 12, volume);
        accessor.Write(position + 16, flags);
        position += 17;
    }

    public void Dispose()
    {
        accessor?.Dispose();
        mmf?.Dispose();
    }
}

// Ticks written directly to disk, no memory pressure
```

## Delta Encoding

```csharp
public class DeltaEncodedTicks
{
    private long baseTimestamp;
    private int basePrice;
    private readonly List<short> timestampDeltas = new();
    private readonly List<short> priceDeltas = new();
    private readonly List<int> volumes = new();

    public void Add(long timestamp, decimal price, int volume)
    {
        if (timestampDeltas.Count == 0)
        {
            baseTimestamp = timestamp;
            basePrice = (int)(price * 10000);
        }
        else
        {
            timestampDeltas.Add((short)(timestamp - baseTimestamp));
            priceDeltas.Add((short)((int)(price * 10000) - basePrice));
        }

        volumes.Add(volume);
    }

    // Much smaller: deltas are 2 bytes instead of 8/4
}
```

## Compressed Tick Format

```csharp
public static class TickCompression
{
    public static byte[] Compress(ReadOnlySpan<CompactTick> ticks)
    {
        using var ms = new MemoryStream();
        using var gz = new GZipStream(ms, CompressionLevel.Fastest);

        var bytes = MemoryMarshal.AsBytes(ticks);
        gz.Write(bytes);

        return ms.ToArray();
    }

    // Typical compression ratio: 3:1 to 5:1
}
```

## Related Patterns

- [Struct and Value Types for Performance](./struct-value-types-performance.md) — Efficient structs
- [Zero-Copy Techniques](./zero-copy-techniques.md) — Avoid copying
- [Memory Safety (Span)](../patterns/memory-safety-span.md) — Stack allocation
- [Market Data Processing](./market-data-processing.md) — Processing pipeline
