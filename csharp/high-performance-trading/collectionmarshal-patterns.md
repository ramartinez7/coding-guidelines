# CollectionMarshal Patterns

> Access collection internals directly for zero-overhead manipulation—bypass bounds checking and intermediate copies using CollectionMarshal APIs.

## Problem

Standard collection APIs create defensive copies, perform bounds checking, and allocate intermediate arrays. CollectionMarshal provides unsafe access to collection internals—getting direct Span<T> references to underlying arrays, enabling zero-copy operations and eliminating overhead in performance-critical code.

## Example

### ❌ Before (Copying and Allocations)

```csharp
public class StandardCollectionAccess
{
    public void ProcessList(List<int> numbers)
    {
        // ❌ Copies to array
        int[] array = numbers.ToArray();
        
        for (int i = 0; i < array.Length; i++)
        {
            array[i] *= 2;
        }
        
        // ❌ Allocates and copies back
        numbers.Clear();
        numbers.AddRange(array);
    }
    
    public void AccessDict(Dictionary<string, int> dict, string key)
    {
        // ❌ Double lookup
        if (dict.ContainsKey(key))
        {
            dict[key] = dict[key] + 1;
        }
    }
}
```

### ✅ After (Direct Access with CollectionMarshal)

```csharp
using System.Runtime.InteropServices;

public class DirectCollectionAccess
{
    public void ProcessList(List<int> numbers)
    {
        // ✅ Direct access to underlying array (zero-copy)
        Span<int> span = CollectionsMarshal.AsSpan(numbers);
        
        for (int i = 0; i < span.Length; i++)
        {
            span[i] *= 2;  // Modifies list directly!
        }
        
        // No copying, no allocations
    }
    
    public void AccessDict(Dictionary<string, int> dict, string key)
    {
        // ✅ Single lookup, get reference to value
        ref int value = ref CollectionsMarshal.GetValueRefOrAddDefault(dict, key, out bool exists);
        
        if (exists)
        {
            value++;  // Modify in place
        }
        else
        {
            value = 1;  // Initialize new entry
        }
    }
}
```

## List Direct Access

```csharp
public class ListMarshalPatterns
{
    // ✅ Get underlying array as Span
    public void DirectSpanAccess(List<int> list)
    {
        Span<int> span = CollectionsMarshal.AsSpan(list);
        
        // Direct memory access, no bounds checking
        for (int i = 0; i < span.Length; i++)
        {
            span[i] = i * i;
        }
    }
    
    // ✅ Use with SIMD
    public void SIMDProcessing(List<float> list)
    {
        Span<float> span = CollectionsMarshal.AsSpan(list);
        
        // Can use with Vector<T> directly
        int vectorSize = Vector<float>.Count;
        
        for (int i = 0; i <= span.Length - vectorSize; i += vectorSize)
        {
            var vector = new Vector<float>(span.Slice(i, vectorSize));
            vector = vector * 2.0f;
            vector.CopyTo(span.Slice(i, vectorSize));
        }
    }
    
    // ✅ Zero-copy sorting
    public void FastSort(List<int> list)
    {
        Span<int> span = CollectionsMarshal.AsSpan(list);
        span.Sort();  // Sorts in place, zero allocations
    }
    
    // ✅ Zero-copy search
    public int BinarySearch(List<int> sortedList, int target)
    {
        ReadOnlySpan<int> span = CollectionsMarshal.AsSpan(sortedList);
        return span.BinarySearch(target);
    }
}
```

## Dictionary Direct Access

```csharp
public class DictionaryMarshalPatterns
{
    // ✅ Get reference to value (or add default)
    public void IncrementCounter(Dictionary<string, int> counters, string key)
    {
        ref int count = ref CollectionsMarshal.GetValueRefOrAddDefault(
            counters, key, out bool exists);
        
        if (exists)
        {
            count++;
        }
        else
        {
            count = 1;
        }
        
        // Single lookup instead of two (ContainsKey + indexer)
    }
    
    // ✅ Accumulate values
    public void AccumulateValues(Dictionary<string, decimal> totals, string key, decimal value)
    {
        ref decimal total = ref CollectionsMarshal.GetValueRefOrAddDefault(
            totals, key, out _);
        
        total += value;  // Modifies directly in dictionary
    }
    
    // ✅ Conditional update
    public void UpdateIfExists(Dictionary<int, OrderState> orders, int orderId, OrderState newState)
    {
        ref OrderState state = ref CollectionsMarshal.GetValueRefOrNullRef(orders, orderId);
        
        if (!Unsafe.IsNullRef(ref state))
        {
            state = newState;  // Update in place
        }
    }
    
    // ✅ Get reference without adding
    public bool TryGetReference(Dictionary<string, int> dict, string key, out int value)
    {
        ref int valueRef = ref CollectionsMarshal.GetValueRefOrNullRef(dict, key);
        
        if (Unsafe.IsNullRef(ref valueRef))
        {
            value = 0;
            return false;
        }
        
        value = valueRef;
        return true;
    }
}

public enum OrderState
{
    Pending,
    Active,
    Filled
}
```

## Market Data Processing

```csharp
public class MarketDataProcessor
{
    private readonly Dictionary<string, Quote> quotes = new();
    private readonly List<decimal> prices = new(10000);
    
    // ✅ Update quote in place
    public void UpdateQuote(string symbol, decimal bid, decimal ask)
    {
        ref Quote quote = ref CollectionsMarshal.GetValueRefOrAddDefault(
            quotes, symbol, out bool exists);
        
        if (exists)
        {
            // Update existing quote
            quote.Bid = bid;
            quote.Ask = ask;
            quote.LastUpdate = DateTime.UtcNow.Ticks;
        }
        else
        {
            // Initialize new quote
            quote = new Quote
            {
                Symbol = symbol,
                Bid = bid,
                Ask = ask,
                LastUpdate = DateTime.UtcNow.Ticks
            };
        }
    }
    
    // ✅ Calculate VWAP with direct access
    public decimal CalculateVWAP()
    {
        Span<decimal> priceSpan = CollectionsMarshal.AsSpan(prices);
        
        decimal sum = 0;
        foreach (decimal price in priceSpan)
        {
            sum += price;
        }
        
        return sum / priceSpan.Length;
    }
    
    // ✅ Process all quotes efficiently
    public void ProcessAllQuotes(Action<Quote> processor)
    {
        // Get direct access to dictionary entries
        foreach (var kvp in quotes)
        {
            ref Quote quote = ref CollectionsMarshal.GetValueRefOrNullRef(quotes, kvp.Key);
            
            if (!Unsafe.IsNullRef(ref quote))
            {
                processor(quote);
            }
        }
    }
}

public struct Quote
{
    public string Symbol { get; set; }
    public decimal Bid { get; set; }
    public decimal Ask { get; set; }
    public long LastUpdate { get; set; }
}
```

## Batch Operations

```csharp
public class BatchOperations
{
    // ✅ Batch update list
    public void BatchUpdate(List<int> list, Func<int, int> transform)
    {
        Span<int> span = CollectionsMarshal.AsSpan(list);
        
        for (int i = 0; i < span.Length; i++)
        {
            span[i] = transform(span[i]);
        }
    }
    
    // ✅ Batch dictionary updates
    public void BatchUpdateCounters(Dictionary<string, int> counters, string[] keys)
    {
        foreach (string key in keys)
        {
            ref int count = ref CollectionsMarshal.GetValueRefOrAddDefault(
                counters, key, out _);
            
            count++;
        }
    }
    
    // ✅ Filter in place
    public void FilterInPlace(List<int> list, Func<int, bool> predicate)
    {
        Span<int> span = CollectionsMarshal.AsSpan(list);
        int writeIndex = 0;
        
        for (int i = 0; i < span.Length; i++)
        {
            if (predicate(span[i]))
            {
                span[writeIndex++] = span[i];
            }
        }
        
        // Truncate list
        list.RemoveRange(writeIndex, list.Count - writeIndex);
    }
}
```

## Statistics Accumulation

```csharp
public class StatisticsAccumulator
{
    private readonly Dictionary<string, Statistics> stats = new();
    
    public void RecordValue(string key, double value)
    {
        ref Statistics stat = ref CollectionsMarshal.GetValueRefOrAddDefault(
            stats, key, out bool exists);
        
        if (!exists)
        {
            stat = new Statistics();
        }
        
        // Update in place (no copying)
        stat.Count++;
        stat.Sum += value;
        stat.Min = exists ? Math.Min(stat.Min, value) : value;
        stat.Max = exists ? Math.Max(stat.Max, value) : value;
    }
    
    public Statistics GetStatistics(string key)
    {
        ref Statistics stat = ref CollectionsMarshal.GetValueRefOrNullRef(stats, key);
        
        if (Unsafe.IsNullRef(ref stat))
        {
            return new Statistics();
        }
        
        return stat;
    }
}

public struct Statistics
{
    public long Count;
    public double Sum;
    public double Min;
    public double Max;
    
    public readonly double Average => Count > 0 ? Sum / Count : 0;
}
```

## Order Book with Direct Access

```csharp
public class OrderBook
{
    private readonly Dictionary<decimal, PriceLevel> bids = new();
    private readonly Dictionary<decimal, PriceLevel> asks = new();
    
    public void AddOrder(decimal price, int quantity, bool isBuy)
    {
        var levels = isBuy ? bids : asks;
        
        ref PriceLevel level = ref CollectionsMarshal.GetValueRefOrAddDefault(
            levels, price, out bool exists);
        
        if (exists)
        {
            // Update existing level
            level.Quantity += quantity;
            level.OrderCount++;
        }
        else
        {
            // Create new level
            level = new PriceLevel
            {
                Price = price,
                Quantity = quantity,
                OrderCount = 1
            };
        }
    }
    
    public void RemoveOrder(decimal price, int quantity, bool isBuy)
    {
        var levels = isBuy ? bids : asks;
        
        ref PriceLevel level = ref CollectionsMarshal.GetValueRefOrNullRef(levels, price);
        
        if (!Unsafe.IsNullRef(ref level))
        {
            level.Quantity -= quantity;
            level.OrderCount--;
            
            if (level.Quantity <= 0)
            {
                levels.Remove(price);
            }
        }
    }
}

public struct PriceLevel
{
    public decimal Price;
    public int Quantity;
    public int OrderCount;
}
```

## Best Practices

### 1. Use with Mutable Structs Carefully

```csharp
// ✅ Ref ensures modifications stick
ref MyStruct value = ref CollectionsMarshal.GetValueRefOrAddDefault(dict, key, out _);
value.Field = 42;  // Modifies dictionary

// ❌ Without ref, modifies copy
MyStruct value = dict[key];
value.Field = 42;  // Doesn't modify dictionary!
```

### 2. Check for Null Ref

```csharp
// ✅ Check before using
ref T value = ref CollectionsMarshal.GetValueRefOrNullRef(dict, key);

if (!Unsafe.IsNullRef(ref value))
{
    // Safe to use value
}
```

### 3. Don't Resize List While Holding Span

```csharp
// ❌ Dangerous: span invalidated if list resizes
Span<int> span = CollectionsMarshal.AsSpan(list);
list.Add(42);  // May reallocate array!
span[0] = 100;  // Undefined behavior!

// ✅ Ensure capacity first
list.EnsureCapacity(requiredSize);
Span<int> span = CollectionsMarshal.AsSpan(list);
```

### 4. Prefer for Hot Paths Only

```csharp
// ✅ Use in performance-critical code
public void HotPath(List<int> list)
{
    Span<int> span = CollectionsMarshal.AsSpan(list);
    // Fast processing
}

// ✅ Use standard APIs elsewhere
public void NormalCode(List<int> list)
{
    foreach (int value in list)  // Fine for normal code
    {
        // ...
    }
}
```

## Performance Characteristics

- **Dictionary single lookup**: 2x faster than Contains + indexer
- **List as Span**: Zero-copy, eliminates ToArray()
- **SIMD with List**: Direct vectorization without copy
- **In-place updates**: Eliminates defensive copies

## When to Use

- High-frequency dictionary updates
- Processing large lists with SIMD
- Accumulating statistics
- Order book implementations
- Zero-allocation requirements

## When NOT to Use

- Concurrent access (not thread-safe)
- Public APIs (too low-level)
- Code clarity more important
- Collection may resize during operation

## Related Patterns

- [MemoryMarshal Advanced](./memorymarshall-advanced.md) — Memory operations
- [Zero-Copy Techniques](./zero-copy-techniques.md) — Span usage
- [Hot Path Optimization](./hot-path-optimization.md) — Performance critical
- [SIMD Vectorization](./simd-vectorization.md) — Vector operations

## References

- "CollectionsMarshal Class" - Microsoft Docs (.NET 5+)
- "High-performance List<T> operations" - .NET Blog
- "ref returns and ref locals" - C# Programming Guide
