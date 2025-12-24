# Market Data Processing

> Process high-frequency market data streams efficiently—handle tick data, aggregate to bars, and maintain multiple timeframes.

## Problem

Market data arrives at extreme rates (millions of ticks per second), and must be parsed, validated, aggregated, and distributed with minimal latency. Traditional event-driven architectures can't keep pace. High-performance market data processing requires zero-copy parsing, lock-free distribution, and efficient aggregation strategies.

## Example

### ❌ Before (Event-Driven, Slow)

```csharp
public class MarketDataProcessor
{
    public event EventHandler<TickEventArgs> OnTick;  // Allocates delegate

    public void ProcessMessage(byte[] data)
    {
        var tick = JsonSerializer.Deserialize<Tick>(data);  // Allocation!
        OnTick?.Invoke(this, new TickEventArgs(tick));  // More allocations!
    }
}

// Problems:
// - Multiple allocations per tick
// - Event handler overhead
// - No backpressure
// - Can't handle high frequency
```

### ✅ After (Zero-Copy Pipeline)

```csharp
public class MarketDataProcessor
{
    private readonly SPSCQueue<Tick> tickQueue;
    private readonly ObjectPool<Tick> tickPool;

    public void ProcessMessage(ReadOnlySpan<byte> data)
    {
        var tick = tickPool.Get();
        
        // Zero-copy parse
        if (TryParseTick(data, tick))
        {
            tickQueue.TryEnqueue(tick);  // Lock-free
        }
        else
        {
            tickPool.Return(tick);
        }
    }

    private bool TryParseTick(ReadOnlySpan<byte> data, Tick tick)
    {
        // Parse directly into pooled object
        // No allocations, no copying
        return true;
    }
}
```

## Tick Aggregation

```csharp
public class TickAggregator
{
    private readonly Dictionary<string, Bar> currentBars = new();
    private readonly TimeSpan barDuration = TimeSpan.FromSeconds(1);

    public void ProcessTick(Tick tick)
    {
        if (!currentBars.TryGetValue(tick.Symbol, out var bar))
        {
            bar = new Bar(tick.Symbol, tick.Timestamp);
            currentBars[tick.Symbol] = bar;
        }

        bar.Update(tick);

        if (tick.Timestamp - bar.StartTime >= barDuration)
        {
            PublishBar(bar);
            currentBars[tick.Symbol] = new Bar(tick.Symbol, tick.Timestamp);
        }
    }
}

public class Bar
{
    public string Symbol { get; }
    public DateTime StartTime { get; }
    public decimal Open { get; private set; }
    public decimal High { get; private set; }
    public decimal Low { get; private set; }
    public decimal Close { get; private set; }
    public long Volume { get; private set; }

    public Bar(string symbol, DateTime startTime)
    {
        Symbol = symbol;
        StartTime = startTime;
    }

    public void Update(Tick tick)
    {
        if (Open == 0) Open = tick.Price;
        High = Math.Max(High, tick.Price);
        Low = Low == 0 ? tick.Price : Math.Min(Low, tick.Price);
        Close = tick.Price;
        Volume += tick.Volume;
    }
}
```

## Multi-Timeframe Aggregation

```csharp
public class MultiTimeframeAggregator
{
    private readonly Dictionary<TimeSpan, BarAggregator> aggregators = new();

    public MultiTimeframeAggregator()
    {
        aggregators[TimeSpan.FromSeconds(1)] = new BarAggregator(TimeSpan.FromSeconds(1));
        aggregators[TimeSpan.FromMinutes(1)] = new BarAggregator(TimeSpan.FromMinutes(1));
        aggregators[TimeSpan.FromMinutes(5)] = new BarAggregator(TimeSpan.FromMinutes(5));
    }

    public void ProcessTick(Tick tick)
    {
        foreach (var aggregator in aggregators.Values)
        {
            aggregator.Update(tick);
        }
    }
}
```

## Level 2 Order Book Updates

```csharp
public readonly struct Level2Update
{
    public string Symbol { get; }
    public Side Side { get; }
    public decimal Price { get; }
    public long Quantity { get; }
    public UpdateType Type { get; }

    // Efficient parsing from FIX or binary
    public static bool TryParse(ReadOnlySpan<byte> data, out Level2Update update)
    {
        // Parse without allocations
        update = default;
        return true;
    }
}

public enum UpdateType { Add, Update, Delete }
```

## Related Patterns

- [Zero-Copy Techniques](./zero-copy-techniques.md) — Avoid data copying
- [Memory Pooling Strategies](./memory-pooling-strategies.md) — Reuse objects
- [Order Book Modeling](./order-book-modeling.md) — Order book structures
- [Tick Data Structures](./tick-data-structures.md) — Efficient tick storage
