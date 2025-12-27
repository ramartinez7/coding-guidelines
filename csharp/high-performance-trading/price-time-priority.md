# Price-Time Priority

> Implement price-time priority matching with type-safe order queues—ensure fair order execution with deterministic matching rules.

## Problem

Order matching engines must enforce price-time priority: orders at better prices execute first, and among same-price orders, earlier orders execute first. Incorrect implementation causes unfair execution, regulatory violations, and lost revenue. Type-safe queue structures ensure priority rules are compiler-enforced.

## Example

### ❌ Before (Unordered List)

```csharp
public class MatchingEngine
{
    private List<Order> buyOrders = new();
    private List<Order> sellOrders = new();

    public void Match(Order incomingOrder)
    {
        // Must manually sort every time
        var orders = incomingOrder.Side == Side.Buy ? sellOrders : buyOrders;
        orders.Sort((a, b) => a.Price.CompareTo(b.Price));  // Wrong! No time priority

        // Price-time priority not guaranteed
    }
}
```

### ✅ After (Price-Time Priority Queue)

```csharp
public class PriceTimePriorityQueue
{
    private readonly SortedDictionary<decimal, Queue<Order>> orders;
    private readonly IComparer<decimal> comparer;

    public PriceTimePriorityQueue(Side side)
    {
        // Buy: descending (highest price first)
        // Sell: ascending (lowest price first)
        comparer = side == Side.Buy
            ? Comparer<decimal>.Create((a, b) => b.CompareTo(a))
            : Comparer<decimal>.Default;

        orders = new SortedDictionary<decimal, Queue<Order>>(comparer);
    }

    public void Add(Order order)
    {
        if (!orders.TryGetValue(order.Price, out var queue))
        {
            queue = new Queue<Order>();
            orders[order.Price] = queue;
        }

        queue.Enqueue(order);  // Time priority within price level
    }

    public Order? GetBest()
    {
        if (orders.Count == 0) return null;

        var firstPrice = orders.First();
        var queue = firstPrice.Value;

        if (queue.Count == 0)
        {
            orders.Remove(firstPrice.Key);
            return GetBest();  // Recursive, find next
        }

        return queue.Dequeue();
    }
}
```

## Matching Engine

```csharp
public class MatchingEngine
{
    private readonly PriceTimePriorityQueue buyOrders;
    private readonly PriceTimePriorityQueue sellOrders;

    public MatchingEngine()
    {
        buyOrders = new PriceTimePriorityQueue(Side.Buy);
        sellOrders = new PriceTimePriorityQueue(Side.Sell);
    }

    public MatchResult Match(Order incomingOrder)
    {
        var result = new MatchResult();

        if (incomingOrder.Side == Side.Buy)
        {
            MatchBuyOrder(incomingOrder, result);
        }
        else
        {
            MatchSellOrder(incomingOrder, result);
        }

        return result;
    }

    private void MatchBuyOrder(Order buy, MatchResult result)
    {
        while (buy.RemainingQuantity > 0)
        {
            var bestSell = sellOrders.GetBest();
            if (bestSell == null || bestSell.Price > buy.Price)
                break;  // No match possible

            var matchQuantity = Math.Min(buy.RemainingQuantity, bestSell.RemainingQuantity);
            var matchPrice = bestSell.Price;  // Maker price

            result.AddFill(buy, bestSell, matchQuantity, matchPrice);

            buy.Fill(matchQuantity);
            bestSell.Fill(matchQuantity);

            if (bestSell.RemainingQuantity > 0)
            {
                sellOrders.Add(bestSell);  // Re-add partial fill
            }
        }

        if (buy.RemainingQuantity > 0)
        {
            buyOrders.Add(buy);  // Resting order
        }
    }
}
```

## Type-Safe Priority

```csharp
// ✅ Type-level enforcement of priority
public readonly struct PriceTimeStamp : IComparable<PriceTimeStamp>
{
    public decimal Price { get; }
    public long Timestamp { get; }
    public Side Side { get; }

    public PriceTimeStamp(decimal price, long timestamp, Side side)
    {
        this.Price = price;
        this.Timestamp = timestamp;
        this.Side = side;
    }

    public int CompareTo(PriceTimeStamp other)
    {
        // Price priority
        var priceCompare = Side == Side.Buy
            ? other.Price.CompareTo(Price)  // Buy: higher is better
            : Price.CompareTo(other.Price); // Sell: lower is better

        if (priceCompare != 0) return priceCompare;

        // Time priority (earlier is better)
        return Timestamp.CompareTo(other.Timestamp);
    }
}
```

## Related Patterns

- [Order Book Modeling](./order-book-modeling.md) — Order book structures
- [Type-Safe Enumerations](../patterns/type-safe-enumerations.md) — Smart enums
- [Making Invalid States Unrepresentable](../patterns/making-invalid-states-unrepresentable.md)
