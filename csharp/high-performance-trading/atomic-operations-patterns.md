# Atomic Operations Patterns

> Use atomic operations for thread-safe updates without locks—leverage CPU's atomic instructions for lock-free counter, flags, and state management.

## Problem

Shared mutable state accessed by multiple threads requires synchronization. Traditional locks add overhead, cause contention, and can deadlock. For simple operations like increments, exchanges, or compare-and-swap, locks are overkill. Modern CPUs provide atomic instructions that guarantee thread-safe updates in a single operation without locks.

## Example

### ❌ Before (Lock-Based)

```csharp
public class LockedCounter
{
    private readonly object lockObj = new();
    private long count;
    
    public void Increment()
    {
        lock (lockObj)  // Context switch, potential contention
        {
            count++;
        }
    }
    
    public long GetValue()
    {
        lock (lockObj)
        {
            return count;
        }
    }
    
    // Problems:
    // - Lock overhead: ~100+ cycles
    // - Contention under high load
    // - No progress guarantee if thread blocked
}
```

### ✅ After (Atomic Operations)

```csharp
public class AtomicCounter
{
    private long count;
    
    public void Increment()
    {
        Interlocked.Increment(ref count);  // ~20-50 cycles, lock-free
    }
    
    public long GetValue()
    {
        return Interlocked.Read(ref count);  // Atomic read
    }
    
    public void Add(long value)
    {
        Interlocked.Add(ref count, value);
    }
    
    // ✅ Lock-free, wait-free, no contention
}
```

## Interlocked Operations

```csharp
public static class InterlockedOperations
{
    // ✅ Atomic increment
    public static long Increment(ref long location)
    {
        return Interlocked.Increment(ref location);
        // Returns value AFTER increment
    }
    
    // ✅ Atomic decrement
    public static long Decrement(ref long location)
    {
        return Interlocked.Decrement(ref location);
        // Returns value AFTER decrement
    }
    
    // ✅ Atomic add
    public static long Add(ref long location, long value)
    {
        return Interlocked.Add(ref location, value);
        // Returns value AFTER addition
    }
    
    // ✅ Atomic exchange (swap)
    public static T Exchange<T>(ref T location, T value) where T : class?
    {
        return Interlocked.Exchange(ref location, value);
        // Returns OLD value, sets NEW value atomically
    }
    
    // ✅ Compare-and-swap (CAS)
    public static T CompareExchange<T>(ref T location, T value, T comparand)
        where T : class?
    {
        return Interlocked.CompareExchange(ref location, value, comparand);
        // If location == comparand: set to value, return OLD value
        // Else: return current value unchanged
    }
    
    // ✅ Atomic read (64-bit on 32-bit systems)
    public static long Read(ref long location)
    {
        return Interlocked.Read(ref location);
        // Ensures atomic 64-bit read on 32-bit platforms
    }
    
    // ✅ Atomic bitwise OR
    public static long Or(ref long location, long value)
    {
        return Interlocked.Or(ref location, value);
        // location |= value atomically
    }
    
    // ✅ Atomic bitwise AND
    public static long And(ref long location, long value)
    {
        return Interlocked.And(ref location, value);
        // location &= value atomically
    }
    
    // ✅ Full memory barrier
    public static void MemoryBarrier()
    {
        Interlocked.MemoryBarrier();
        // Prevents instruction reordering
    }
}
```

## Compare-and-Swap (CAS) Patterns

```csharp
public class CASPatterns
{
    // ✅ CAS-based increment (alternative to Interlocked.Increment)
    public static long CASIncrement(ref long location)
    {
        long current, newValue;
        
        do
        {
            current = Interlocked.Read(ref location);
            newValue = current + 1;
        }
        while (Interlocked.CompareExchange(ref location, newValue, current) != current);
        
        return newValue;
    }
    
    // ✅ CAS-based maximum
    public static void UpdateMax(ref long location, long value)
    {
        long current;
        
        do
        {
            current = Interlocked.Read(ref location);
            
            if (value <= current)
                return;  // Not greater, no update needed
        }
        while (Interlocked.CompareExchange(ref location, value, current) != current);
    }
    
    // ✅ CAS-based minimum
    public static void UpdateMin(ref long location, long value)
    {
        long current;
        
        do
        {
            current = Interlocked.Read(ref location);
            
            if (value >= current)
                return;
        }
        while (Interlocked.CompareExchange(ref location, value, current) != current);
    }
    
    // ✅ Conditional update with CAS
    public static bool TryUpdate(ref long location, Func<long, long> update, Func<long, bool> condition)
    {
        long current, newValue;
        
        do
        {
            current = Interlocked.Read(ref location);
            
            if (!condition(current))
                return false;
            
            newValue = update(current);
        }
        while (Interlocked.CompareExchange(ref location, newValue, current) != current);
        
        return true;
    }
}
```

## Atomic Reference Updates

```csharp
public class AtomicReference<T> where T : class
{
    private T value;
    
    public AtomicReference(T initialValue)
    {
        this.value = initialValue;
    }
    
    // ✅ Get current value
    public T Get()
    {
        return Volatile.Read(ref value);
    }
    
    // ✅ Set value atomically
    public void Set(T newValue)
    {
        Volatile.Write(ref value, newValue);
    }
    
    // ✅ Exchange (swap)
    public T GetAndSet(T newValue)
    {
        return Interlocked.Exchange(ref value, newValue);
    }
    
    // ✅ Compare-and-swap
    public bool CompareAndSet(T expected, T newValue)
    {
        return Interlocked.CompareExchange(ref value, newValue, expected) == expected;
    }
    
    // ✅ Update with function
    public T UpdateAndGet(Func<T, T> updateFunc)
    {
        T current, newValue;
        
        do
        {
            current = Volatile.Read(ref value);
            newValue = updateFunc(current);
        }
        while (Interlocked.CompareExchange(ref value, newValue, current) != current);
        
        return newValue;
    }
    
    // ✅ Get and update
    public T GetAndUpdate(Func<T, T> updateFunc)
    {
        T current, newValue;
        
        do
        {
            current = Volatile.Read(ref value);
            newValue = updateFunc(current);
        }
        while (Interlocked.CompareExchange(ref value, newValue, current) != current);
        
        return current;  // Return OLD value
    }
}
```

## Atomic Flags

```csharp
public class AtomicBoolean
{
    private int value;  // 0 = false, 1 = true
    
    public AtomicBoolean(bool initialValue = false)
    {
        this.value = initialValue ? 1 : 0;
    }
    
    // ✅ Get value
    public bool Get()
    {
        return Interlocked.Read(ref value) != 0;
    }
    
    // ✅ Set value
    public void Set(bool newValue)
    {
        Interlocked.Exchange(ref value, newValue ? 1 : 0);
    }
    
    // ✅ Test-and-set (get old value, set to true)
    public bool GetAndSet(bool newValue)
    {
        int oldValue = Interlocked.Exchange(ref value, newValue ? 1 : 0);
        return oldValue != 0;
    }
    
    // ✅ Compare-and-set
    public bool CompareAndSet(bool expected, bool newValue)
    {
        int expectedInt = expected ? 1 : 0;
        int newInt = newValue ? 1 : 0;
        return Interlocked.CompareExchange(ref value, newInt, expectedInt) == expectedInt;
    }
    
    // ✅ Test-and-set flag (common pattern)
    public bool TrySet()
    {
        return Interlocked.CompareExchange(ref value, 1, 0) == 0;
    }
    
    // ✅ Test-and-clear
    public bool TryClear()
    {
        return Interlocked.CompareExchange(ref value, 0, 1) == 1;
    }
}
```

## Order Book with Atomic Operations

```csharp
public class AtomicOrderBook
{
    private long bestBidPrice;  // Fixed-point representation
    private long bestAskPrice;
    private long lastTradePrice;
    private long volumeTraded;
    
    // ✅ Update best bid (only if higher)
    public void UpdateBestBid(decimal price)
    {
        long priceLong = (long)(price * 10000);  // Fixed-point
        
        long current;
        do
        {
            current = Interlocked.Read(ref bestBidPrice);
            
            if (priceLong <= current)
                return;  // Not better
        }
        while (Interlocked.CompareExchange(ref bestBidPrice, priceLong, current) != current);
    }
    
    // ✅ Update best ask (only if lower)
    public void UpdateBestAsk(decimal price)
    {
        long priceLong = (long)(price * 10000);
        
        long current;
        do
        {
            current = Interlocked.Read(ref bestAskPrice);
            
            if (priceLong >= current && current != 0)
                return;  // Not better
        }
        while (Interlocked.CompareExchange(ref bestAskPrice, priceLong, current) != current);
    }
    
    // ✅ Record trade
    public void RecordTrade(decimal price, int volume)
    {
        long priceLong = (long)(price * 10000);
        
        Interlocked.Exchange(ref lastTradePrice, priceLong);
        Interlocked.Add(ref volumeTraded, volume);
    }
    
    // ✅ Get snapshot atomically
    public (decimal BestBid, decimal BestAsk, decimal LastTrade, long Volume) GetSnapshot()
    {
        return (
            Interlocked.Read(ref bestBidPrice) / 10000m,
            Interlocked.Read(ref bestAskPrice) / 10000m,
            Interlocked.Read(ref lastTradePrice) / 10000m,
            Interlocked.Read(ref volumeTraded)
        );
    }
}
```

## Statistics Counters

```csharp
public class AtomicStatistics
{
    private long count;
    private long sum;
    private long min = long.MaxValue;
    private long max = long.MinValue;
    
    // ✅ Record value
    public void Record(long value)
    {
        Interlocked.Increment(ref count);
        Interlocked.Add(ref sum, value);
        
        // Update min
        long currentMin;
        do
        {
            currentMin = Interlocked.Read(ref min);
            
            if (value >= currentMin)
                break;
        }
        while (Interlocked.CompareExchange(ref min, value, currentMin) != currentMin);
        
        // Update max
        long currentMax;
        do
        {
            currentMax = Interlocked.Read(ref max);
            
            if (value <= currentMax)
                break;
        }
        while (Interlocked.CompareExchange(ref max, value, currentMax) != currentMax);
    }
    
    // ✅ Get statistics
    public (long Count, double Average, long Min, long Max) GetStats()
    {
        long c = Interlocked.Read(ref count);
        long s = Interlocked.Read(ref sum);
        long mn = Interlocked.Read(ref min);
        long mx = Interlocked.Read(ref max);
        
        double avg = c > 0 ? (double)s / c : 0;
        
        return (c, avg, mn, mx);
    }
    
    // ✅ Reset
    public void Reset()
    {
        Interlocked.Exchange(ref count, 0);
        Interlocked.Exchange(ref sum, 0);
        Interlocked.Exchange(ref min, long.MaxValue);
        Interlocked.Exchange(ref max, long.MinValue);
    }
}
```

## Rate Limiter with Atomic Operations

```csharp
public class AtomicRateLimiter
{
    private long tokenCount;
    private long lastRefillTicks;
    private readonly long maxTokens;
    private readonly long refillIntervalTicks;
    private readonly long tokensPerInterval;
    
    public AtomicRateLimiter(long maxTokens, TimeSpan refillInterval, long tokensPerInterval)
    {
        this.maxTokens = maxTokens;
        this.refillIntervalTicks = refillInterval.Ticks;
        this.tokensPerInterval = tokensPerInterval;
        this.tokenCount = maxTokens;
        this.lastRefillTicks = DateTime.UtcNow.Ticks;
    }
    
    // ✅ Try acquire token
    public bool TryAcquire()
    {
        RefillTokens();
        
        long current;
        do
        {
            current = Interlocked.Read(ref tokenCount);
            
            if (current <= 0)
                return false;  // No tokens available
        }
        while (Interlocked.CompareExchange(ref tokenCount, current - 1, current) != current);
        
        return true;
    }
    
    private void RefillTokens()
    {
        long now = DateTime.UtcNow.Ticks;
        long last = Interlocked.Read(ref lastRefillTicks);
        long elapsed = now - last;
        
        if (elapsed < refillIntervalTicks)
            return;
        
        // Try to update last refill time
        if (Interlocked.CompareExchange(ref lastRefillTicks, now, last) != last)
            return;  // Another thread did the refill
        
        // Add tokens
        long tokensToAdd = (elapsed / refillIntervalTicks) * tokensPerInterval;
        
        long current;
        do
        {
            current = Interlocked.Read(ref tokenCount);
            long newCount = Math.Min(current + tokensToAdd, maxTokens);
            
            if (newCount == current)
                break;
            
            if (Interlocked.CompareExchange(ref tokenCount, newCount, current) == current)
                break;
        }
        while (true);
    }
}
```

## Atomic State Machine

```csharp
public enum OrderState
{
    Pending = 0,
    Active = 1,
    PartiallyFilled = 2,
    Filled = 3,
    Cancelled = 4,
    Rejected = 5
}

public class AtomicStateMachine
{
    private int state = (int)OrderState.Pending;
    
    // ✅ Valid state transitions
    private static readonly Dictionary<OrderState, OrderState[]> ValidTransitions = new()
    {
        [OrderState.Pending] = new[] { OrderState.Active, OrderState.Rejected },
        [OrderState.Active] = new[] { OrderState.PartiallyFilled, OrderState.Filled, OrderState.Cancelled },
        [OrderState.PartiallyFilled] = new[] { OrderState.Filled, OrderState.Cancelled },
        [OrderState.Filled] = Array.Empty<OrderState>(),
        [OrderState.Cancelled] = Array.Empty<OrderState>(),
        [OrderState.Rejected] = Array.Empty<OrderState>()
    };
    
    // ✅ Try transition
    public bool TryTransition(OrderState newState)
    {
        int current;
        
        do
        {
            current = Interlocked.Read(ref state);
            var currentState = (OrderState)current;
            
            // Check if transition is valid
            if (!ValidTransitions[currentState].Contains(newState))
                return false;
        }
        while (Interlocked.CompareExchange(ref state, (int)newState, current) != current);
        
        return true;
    }
    
    // ✅ Get current state
    public OrderState GetState()
    {
        return (OrderState)Interlocked.Read(ref state);
    }
}
```

## Best Practices

### 1. Use Appropriate Interlocked Method

```csharp
// ✅ Use specific method when available
Interlocked.Increment(ref counter);  // Not CAS loop

// ✅ Use CAS for complex updates
long current, newValue;
do
{
    current = value;
    newValue = ComputeNewValue(current);
}
while (Interlocked.CompareExchange(ref value, newValue, current) != current);
```

### 2. Handle CAS Failures with Backoff

```csharp
// ✅ Exponential backoff on contention
int retries = 0;
while (Interlocked.CompareExchange(ref location, newValue, expected) != expected)
{
    if (retries++ > 100)
        Thread.Yield();  // Reduce contention
    
    expected = Interlocked.Read(ref location);
    newValue = ComputeNewValue(expected);
}
```

### 3. Use Memory Barriers When Needed

```csharp
// ✅ Barrier for visibility
field1 = value1;
field2 = value2;
Interlocked.MemoryBarrier();  // Ensure writes visible
```

### 4. Avoid ABA Problem

```csharp
// ❌ ABA problem: value changes A→B→A, CAS succeeds incorrectly

// ✅ Use version counter
private long value;
private long version;

public void Update(long newValue)
{
    long currentValue, currentVersion;
    do
    {
        currentValue = Interlocked.Read(ref value);
        currentVersion = Interlocked.Read(ref version);
    }
    while (Interlocked.CompareExchange(ref value, newValue, currentValue) != currentValue);
    
    Interlocked.Increment(ref version);
}
```

## Performance Characteristics

- **Interlocked.Increment**: 20-50 cycles
- **Interlocked.CompareExchange**: 20-100 cycles
- **Lock**: 100+ cycles minimum
- **Wait-free**: Guaranteed completion
- **Lock-free**: System-wide progress

## When to Use

- Simple counters and flags
- Lock-free data structures
- State machines with few states
- Statistics gathering
- Rate limiting

## When NOT to Use

- Complex multi-field updates (use locks)
- Need for fairness (Interlocked doesn't guarantee)
- High contention (consider locks with queuing)

## Related Patterns

- [Compare and Swap Explained](./compare-and-swap-explained.md) — CAS deep dive
- [Lock-Free Data Structures](./lock-free-data-structures.md) — Lock-free patterns
- [Memory Barriers](./memory-barriers-synchronization.md) — Memory ordering
- [Volatile Explained](./volatile-explained.md) — Volatile semantics

## References

- ".NET Interlocked Class" - Microsoft Docs
- "The Art of Multiprocessor Programming" by Herlihy & Shavit
- "Java Concurrency in Practice" (concepts apply to C#)
