# Hot Path Optimization

> Identify and optimize the critical execution path—remove allocations, branches, and virtual calls from hot loops.

## Problem

Not all code is equal. 10% of code executes 90% of the time. The "hot path" is the critical execution path that runs millions of times per second. Optimizing cold code wastes time, while unoptimized hot paths kill performance. Profile to find hot paths, then eliminate every source of overhead.

## Example

### ❌ Before (Unoptimized Hot Path)

```csharp
public class OrderMatcher
{
    private readonly ILogger logger;
    
    public MatchResult Match(Order order)
    {
        // This runs 1M times/sec!
        
        logger.LogDebug($"Matching order {order.Id}");  // Allocation!
        
        var bestPrice = GetBestPrice(order.Symbol);  // Virtual call
        
        if (order.Price >= bestPrice)  // Unpredictable branch
        {
            var result = new MatchResult  // Allocation!
            {
                Matched = true,
                Price = bestPrice
            };
            return result;
        }
        
        return MatchResult.NoMatch;
    }
}
```

### ✅ After (Optimized Hot Path)

```csharp
public class OrderMatcher
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public MatchResult Match(ref Order order, decimal bestPrice)
    {
        // Removed: logging (do in caller if needed)
        // Removed: virtual call (pass price directly)
        // Removed: allocation (use pre-allocated result)
        
        if (order.Price >= bestPrice)
        {
            return new MatchResult(true, bestPrice);  // Struct, no allocation
        }
        
        return MatchResult.NoMatch;
    }
}
```

## Profile to Find Hot Paths

```csharp
// Use BenchmarkDotNet or profiler to identify hot methods
[MemoryDiagnoser]
public class OrderMatchingBenchmark
{
    private OrderMatcher matcher;
    private Order[] orders;

    [Benchmark]
    public void MatchOrders()
    {
        for (int i = 0; i < orders.Length; i++)
        {
            matcher.Match(ref orders[i], 100m);  // This is hot!
        }
    }
}
```

## Inline Hot Methods

```csharp
// ✅ Force inlining for small hot methods
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public bool IsValid(Order order)
{
    return order.Price > 0 && order.Quantity > 0;
}

// Compiler inlines call, eliminates call overhead
```

## Eliminate Allocations

```csharp
// ❌ Allocates every call
public string FormatOrder(Order order)
{
    return $"Order {order.Id}: {order.Symbol} @ {order.Price}";
}

// ✅ Use struct result, no allocation
public OrderSummary GetSummary(Order order)
{
    return new OrderSummary(order.Id, order.Symbol, order.Price);
}

// ✅ Or use Span for formatting
public void FormatOrder(Order order, Span<char> destination)
{
    // Write directly to destination span
}
```

## Avoid Virtual Calls

```csharp
// ❌ Virtual call in hot path
public interface IOrderValidator
{
    bool Validate(Order order);
}

public class OrderProcessor
{
    private readonly IOrderValidator validator;
    
    public void Process(Order order)
    {
        if (validator.Validate(order))  // Virtual dispatch!
        {
            // ...
        }
    }
}

// ✅ Use concrete type or generic
public class OrderProcessor<TValidator> 
    where TValidator : IOrderValidator
{
    private readonly TValidator validator;
    
    public void Process(Order order)
    {
        if (validator.Validate(order))  // Devirtualized!
        {
            // ...
        }
    }
}
```

## Minimize Branch Mispredictions

```csharp
// ❌ Unpredictable branches
public void ProcessOrders(Order[] orders)
{
    for (int i = 0; i < orders.Length; i++)
    {
        if (Random.Shared.NextDouble() > 0.5)  // Unpredictable!
        {
            ProcessBuy(orders[i]);
        }
        else
        {
            ProcessSell(orders[i]);
        }
    }
}

// ✅ Predictable branches
public void ProcessOrders(Order[] orders)
{
    for (int i = 0; i < orders.Length; i++)
    {
        if (orders[i].Type == OrderType.Buy)  // More predictable
        {
            ProcessBuy(orders[i]);
        }
        else
        {
            ProcessSell(orders[i]);
        }
    }
}

// ✅ Or eliminate branch entirely
public void ProcessOrders(Order[] orders)
{
    for (int i = 0; i < orders.Length; i++)
    {
        orders[i].Process();  // Virtual call more predictable than random branch
    }
}
```

## Use Structs for Small Data

```csharp
// ✅ Struct for hot path data
public readonly struct MatchResult
{
    public bool Matched { get; }
    public decimal Price { get; }
    
    public MatchResult(bool matched, decimal price)
    {
        Matched = matched;
        Price = price;
    }
}
```

## Best Practices

### 1. Remove Logging from Hot Path

```csharp
// ❌ Logging in hot path
public void Process(Order order)
{
    logger.LogDebug($"Processing {order.Id}");  // Allocation + I/O
    // ...
}

// ✅ Move logging outside or use conditional
public void Process(Order order)
{
    // ... process ...
    
    if (logger.IsEnabled(LogLevel.Debug))
    {
        logger.LogDebug($"Processed {order.Id}");  // Outside hot path
    }
}
```

### 2. Pass by Ref for Large Structs

```csharp
// ✅ Pass by ref to avoid copy
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public void Process(ref Order order)
{
    // No struct copy
}
```

### 3. Use Stackalloc for Small Buffers

```csharp
// ✅ Stack allocation in hot path
Span<byte> buffer = stackalloc byte[256];
```

## Related Patterns

- [Latency-Sensitive Design](./latency-sensitive-design.md)
- [Allocation Budget](../patterns/allocation-budget.md)
- [Memory Safety (Span)](../patterns/memory-safety-span.md)
