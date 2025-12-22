# Busy-Wait vs Sleep

> Choose the right waiting strategy for your latency requirements—busy-wait for ultra-low latency, sleep for efficiency.

## Problem

When waiting for events, you must choose between busy-waiting (consuming CPU cycles) and sleeping (adding latency). Busy-waiting provides sub-microsecond response times but wastes CPU. Sleeping saves CPU but adds milliseconds of latency. The choice depends on your latency requirements and system resources.

## Example

### ❌ Sleep in Latency-Critical Code

```csharp
// ❌ Sleep adds unpredictable latency
while (!IsDataAvailable())
{
    Thread.Sleep(1);  // Minimum 1ms, actually 10-15ms!
}
```

### ✅ Busy-Wait for Low Latency

```csharp
// ✅ Busy-wait for sub-microsecond latency
while (!IsDataAvailable())
{
    Thread.SpinWait(100);  // ~100ns, no context switch
}
```

## Hybrid Approach

```csharp
public void WaitForData()
{
    int spinCount = 0;
    const int MaxSpins = 1000;
    
    while (!IsDataAvailable())
    {
        if (spinCount++ < MaxSpins)
        {
            Thread.SpinWait(10);  // Spin first
        }
        else
        {
            Thread.Sleep(0);  // Yield after spinning
        }
    }
}
```

## When to Use

- **Busy-wait**: Ultra-low latency (<1μs), dedicated CPU cores
- **Sleep**: General purpose, shared resources
- **Hybrid**: Balance latency and CPU efficiency

## Related Patterns

- [Latency-Sensitive Design](./latency-sensitive-design.md)
- [Thread Affinity](./thread-affinity.md)
- [Predictable Performance](./predictable-performance.md)
