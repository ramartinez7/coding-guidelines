# Wait-Free Algorithms

> Guarantee bounded completion time without blocking—use wait-free data structures for deterministic performance.

## Problem

Lock-free algorithms guarantee system-wide progress but individual threads can starve under contention. Wait-free algorithms guarantee every operation completes in bounded time, providing deterministic worst-case performance crucial for real-time trading systems.

## Example

### ❌ Before (Lock-Free but Not Wait-Free)

```csharp
// Lock-free but can retry indefinitely under contention
public class LockFreeCounter
{
    private long value;
    
    public void Increment()
    {
        while (true)  // Unbounded retries!
        {
            long current = Volatile.Read(ref value);
            long next = current + 1;
            
            if (Interlocked.CompareExchange(ref value, next, current) == current)
                return;
                
            // Retry if failed - could loop forever under high contention
        }
    }
}
```

### ✅ After (Wait-Free)

```csharp
// Wait-free: guaranteed completion
public class WaitFreeCounter
{
    private long value;
    
    public void Increment()
    {
        Interlocked.Increment(ref value);  // Wait-free primitive
        // Always completes in bounded time
    }
}
```

## Wait-Free Read

```csharp
// ✅ Simple wait-free read
public class WaitFreeReader
{
    private long value;
    
    public long Read()
    {
        return Volatile.Read(ref value);  // Wait-free
    }
    
    public void Write(long newValue)
    {
        Volatile.Write(ref value, newValue);  // Wait-free
    }
}
```

## Wait-Free SPSC Queue

```csharp
// Single Producer Single Consumer wait-free queue
public class WaitFreeSPSCQueue<T>
{
    private readonly T[] buffer;
    private readonly int mask;
    private long head;
    private long tail;
    
    public WaitFreeSPSCQueue(int capacity)
    {
        if ((capacity & (capacity - 1)) != 0)
            throw new ArgumentException("Capacity must be power of 2");
            
        buffer = new T[capacity];
        mask = capacity - 1;
    }
    
    // Wait-free enqueue (producer only)
    public bool TryEnqueue(T item)
    {
        long currentTail = Volatile.Read(ref tail);
        long nextTail = currentTail + 1;
        
        if (nextTail - Volatile.Read(ref head) > buffer.Length)
            return false;  // Queue full
            
        buffer[currentTail & mask] = item;
        Volatile.Write(ref tail, nextTail);
        return true;
    }
    
    // Wait-free dequeue (consumer only)
    public bool TryDequeue(out T item)
    {
        long currentHead = Volatile.Read(ref head);
        
        if (currentHead >= Volatile.Read(ref tail))
        {
            item = default!;
            return false;  // Queue empty
        }
        
        item = buffer[currentHead & mask];
        Volatile.Write(ref head, currentHead + 1);
        return true;
    }
}
```

## Related Patterns

- [Lock-Free Data Structures](./lock-free-data-structures.md)
- [Lock-Free Queues](./lock-free-queues.md)
- [Predictable Performance](./predictable-performance.md)
