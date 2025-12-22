# Predictable Performance

> Build systems with consistent performance characteristics—avoid GC pauses, JIT compilation delays, and OS scheduling variability.

## Problem

Performance unpredictability kills trading systems. A system that processes most orders in 10μs but occasionally takes 100ms is worse than one consistently taking 50μs. Predictability matters more than peak performance because unpredictable spikes cause missed trades and losses.

## Example

### ❌ Unpredictable Performance

```csharp
public class OrderProcessor
{
    public void ProcessOrder(Order order)
    {
        // GC pause: 0-50ms
        // JIT compilation: 0-100ms on first call
        // Context switch: 0-10ms
        // Thread pool delay: 0-5ms
        
        // Result: p50=10μs, p99=100ms (unusable!)
    }
}
```

### ✅ Predictable Performance

```csharp
public class OrderProcessor
{
    // Pre-allocate everything at startup
    private readonly ObjectPool<Order> pool;
    
    public OrderProcessor()
    {
        // Force JIT compilation
        RuntimeHelpers.PrepareMethod(
            typeof(OrderProcessor).GetMethod(nameof(ProcessOrder))!.MethodHandle);
            
        // Warmup
        for (int i = 0; i < 10000; i++)
        {
            var dummy = pool.Get();
            ProcessOrder(dummy);
            pool.Return(dummy);
        }
    }
    
    [MethodImpl(MethodImplOptions.NoInlining)]
    public void ProcessOrder(Order order)
    {
        // Zero allocations: no GC
        // Already JIT'd: no compilation delay
        // Dedicated thread: no scheduling
        
        // Result: p50=10μs, p99=15μs (consistent!)
    }
}
```

## Eliminate GC Variability

```csharp
// ✅ Disable GC in critical section
GC.TryStartNoGCRegion(1024 * 1024);
try
{
    ProcessCriticalWorkload();
}
finally
{
    GC.EndNoGCRegion();
}
```

## Pre-JIT Compilation

```csharp
// ✅ Force JIT at startup
public static void PrepareMethod<T>(Expression<Action<T>> method)
{
    var methodInfo = ((MethodCallExpression)method.Body).Method;
    RuntimeHelpers.PrepareMethod(methodInfo.MethodHandle);
}

// Usage
PrepareMethod<OrderProcessor>(p => p.ProcessOrder(default!));
```

## Use Dedicated Threads

```csharp
// ✅ Dedicated thread, no thread pool delays
var thread = new Thread(ProcessLoop)
{
    Priority = ThreadPriority.Highest,
    IsBackground = false
};
thread.Start();
```

## Related Patterns

- [Latency-Sensitive Design](./latency-sensitive-design.md)
- [Thread Affinity](./thread-affinity.md)
- [Allocation Budget](../patterns/allocation-budget.md)
