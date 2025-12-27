# Order Book Modeling

> Model order books with type-safe price levels and efficient updates—represent bid/ask spreads and maintain price-time priority.

## Problem

Order books are the core data structure in trading systems, maintaining buy and sell orders organized by price and time. Traditional implementations using dictionaries or lists are too slow for high-frequency trading. Order books must support microsecond updates, efficient price level lookups, and maintain price-time priority.

## Example

### ❌ Before (Dictionary-Based)

```csharp
public class OrderBook
{
    private Dictionary<decimal, List<Order>> bids = new();
    private Dictionary<decimal, List<Order>> asks = new();

    public void AddOrder(Order order)
    {
        var side = order.Side == Side.Buy ? bids : asks;
        if (!side.ContainsKey(order.Price))
            side[order.Price] = new List<Order>();  // Allocation!

        side[order.Price].Add(order);  // Not price-time sorted
    }

    public decimal GetBestBid() => bids.Keys.Max();  // O(n) scan!
}
```

### ✅ After (Efficient Price Levels)

```csharp
public readonly struct PriceLevel
{
    public decimal Price { get; }
    public long TotalVolume { get; }
    public int OrderCount { get; }

    public PriceLevel(decimal price, long totalVolume, int orderCount)
    {
        this.Price = price;
        this.TotalVolume = totalVolume;
        this.OrderCount = orderCount;
    }
}

public class OrderBook
{
    private readonly SortedDictionary<decimal, Queue<Order>> bids;
    private readonly SortedDictionary<decimal, Queue<Order>> asks;
    private decimal bestBid;
    private decimal bestAsk;

    public OrderBook()
    {
        bids = new SortedDictionary<decimal, Queue<Order>>(
            Comparer<decimal>.Create((a, b) => b.CompareTo(a)));  // Descending
        asks = new SortedDictionary<decimal, Queue<Order>>();  // Ascending
    }

    public void AddOrder(Order order)
    {
        var side = order.Side == Side.Buy ? bids : asks;

        if (!side.TryGetValue(order.Price, out var queue))
        {
            queue = new Queue<Order>();
            side[order.Price] = queue;
        }

        queue.Enqueue(order);  // Price-time priority

        if (order.Side == Side.Buy && order.Price > bestBid)
            bestBid = order.Price;
        else if (order.Side == Side.Sell && order.Price < bestAsk)
            bestAsk = order.Price;
    }

    public decimal GetBestBid() => bestBid;  // O(1)
    public decimal GetBestAsk() => bestAsk;  // O(1)
    public decimal GetSpread() => bestAsk - bestBid;
}

public enum Side { Buy, Sell }
```

## Market Depth

```csharp
public class MarketDepth
{
    private readonly PriceLevel[] bidLevels;
    private readonly PriceLevel[] askLevels;

    public MarketDepth(int maxDepth = 10)
    {
        bidLevels = new PriceLevel[maxDepth];
        askLevels = new PriceLevel[maxDepth];
    }

    public ReadOnlySpan<PriceLevel> BidLevels => bidLevels.AsSpan();
    public ReadOnlySpan<PriceLevel> AskLevels => askLevels.AsSpan();
}
```

## Type-Safe Orders

```csharp
public abstract record OrderState
{
    public sealed record Pending : OrderState;
    public sealed record PartiallyFilled(long FilledQuantity) : OrderState;
    public sealed record Filled : OrderState;
    public sealed record Cancelled : OrderState;

    private OrderState() { }
}

public sealed class Order
{
    public OrderId Id { get; }
    public string Symbol { get; }
    public Side Side { get; }
    public decimal Price { get; }
    public long Quantity { get; }
    public OrderState State { get; private set; }

    public void Fill(long quantity)
    {
        State = quantity == Quantity
            ? new OrderState.Filled()
            : new OrderState.PartiallyFilled(quantity);
    }
}
```

## Related Patterns

- [Price-Time Priority](./price-time-priority.md) — Order matching rules
- [Market Data Processing](./market-data-processing.md) — Data stream handling
- [Lock-Free Data Structures](./lock-free-data-structures.md) — Concurrent updates
- [Memory Pooling Strategies](./memory-pooling-strategies.md) — Reduce allocations
