# Lock-Free Queues

> Implement high-throughput producer-consumer queues without locks—use ring buffers and atomic operations for maximum performance.

## Problem

Traditional lock-based queues create bottlenecks in high-frequency systems. When multiple producers and consumers compete for locks, throughput drops and latencies spike. Lock-free queues using atomic operations and ring buffers eliminate contention, achieving millions of operations per second with consistent latency.

## Example

### ❌ Before (Lock-Based Queue)

```csharp
public class LockedQueue<T>
{
    private readonly object lockObj = new();
    private readonly Queue<T> queue = new();

    public void Enqueue(T item)
    {
        lock (lockObj)  // Contention point
        {
            queue.Enqueue(item);
        }
    }

    public bool TryDequeue(out T item)
    {
        lock (lockObj)  // All operations serialize
        {
            if (queue.Count == 0)
            {
                item = default!;
                return false;
            }

            item = queue.Dequeue();
            return true;
        }
    }
}

// Problems:
// - Lock contention limits throughput
// - Unpredictable latencies
// - Poor cache behavior
```

### ✅ After (Lock-Free Ring Buffer Queue)

```csharp
public class LockFreeQueue<T>
{
    private readonly T[] buffer;
    private readonly int mask;
    private long head;
    private long tail;

    public LockFreeQueue(int capacity)
    {
        if ((capacity & (capacity - 1)) != 0)
            throw new ArgumentException("Capacity must be power of 2");

        buffer = new T[capacity];
        mask = capacity - 1;
    }

    public bool TryEnqueue(T item)
    {
        while (true)
        {
            long currentTail = Volatile.Read(ref tail);
            long currentHead = Volatile.Read(ref head);

            if (currentTail - currentHead >= buffer.Length)
                return false;  // Queue full

            if (Interlocked.CompareExchange(ref tail, currentTail + 1, currentTail) == currentTail)
            {
                buffer[currentTail & mask] = item;
                return true;
            }
        }
    }

    public bool TryDequeue(out T item)
    {
        while (true)
        {
            long currentHead = Volatile.Read(ref head);
            long currentTail = Volatile.Read(ref tail);

            if (currentHead >= currentTail)
            {
                item = default!;
                return false;  // Queue empty
            }

            if (Interlocked.CompareExchange(ref head, currentHead + 1, currentHead) == currentHead)
            {
                item = buffer[currentHead & mask];
                return true;
            }
        }
    }
}
```

## Single Producer Single Consumer (SPSC) Queue

```csharp
// Even faster: SPSC removes atomic operations
public class SPSCQueue<T>
{
    private readonly T[] buffer;
    private readonly int mask;
    private long head;  // Only consumer writes
    private long tail;  // Only producer writes

    public SPSCQueue(int capacity)
    {
        if ((capacity & (capacity - 1)) != 0)
            throw new ArgumentException("Capacity must be power of 2");

        buffer = new T[capacity];
        mask = capacity - 1;
    }

    // Producer only
    public bool TryEnqueue(T item)
    {
        long currentTail = tail;
        long nextTail = currentTail + 1;

        if (nextTail - Volatile.Read(ref head) > buffer.Length)
            return false;

        buffer[currentTail & mask] = item;
        Volatile.Write(ref tail, nextTail);
        return true;
    }

    // Consumer only
    public bool TryDequeue(out T item)
    {
        long currentHead = head;

        if (currentHead >= Volatile.Read(ref tail))
        {
            item = default!;
            return false;
        }

        item = buffer[currentHead & mask];
        Volatile.Write(ref head, currentHead + 1);
        return true;
    }
}
```

## Bounded MPMC Queue

```csharp
// Multiple Producer Multiple Consumer with bounded capacity
public class MPMCQueue<T>
{
    private readonly Cell<T>[] buffer;
    private readonly int mask;
    private long enqueuePos;
    private long dequeuePos;

    public MPMCQueue(int capacity)
    {
        if ((capacity & (capacity - 1)) != 0)
            throw new ArgumentException("Capacity must be power of 2");

        buffer = new Cell<T>[capacity];
        mask = capacity - 1;

        for (int i = 0; i < capacity; i++)
        {
            buffer[i] = new Cell<T> { Sequence = i };
        }
    }

    public bool TryEnqueue(T item)
    {
        while (true)
        {
            long pos = Volatile.Read(ref enqueuePos);
            ref var cell = ref buffer[pos & mask];
            long seq = Volatile.Read(ref cell.Sequence);
            long diff = seq - pos;

            if (diff == 0)
            {
                if (Interlocked.CompareExchange(ref enqueuePos, pos + 1, pos) == pos)
                {
                    cell.Value = item;
                    Volatile.Write(ref cell.Sequence, pos + 1);
                    return true;
                }
            }
            else if (diff < 0)
            {
                return false;  // Queue full
            }
        }
    }

    public bool TryDequeue(out T item)
    {
        while (true)
        {
            long pos = Volatile.Read(ref dequeuePos);
            ref var cell = ref buffer[pos & mask];
            long seq = Volatile.Read(ref cell.Sequence);
            long diff = seq - (pos + 1);

            if (diff == 0)
            {
                if (Interlocked.CompareExchange(ref dequeuePos, pos + 1, pos) == pos)
                {
                    item = cell.Value!;
                    Volatile.Write(ref cell.Sequence, pos + mask + 1);
                    return true;
                }
            }
            else if (diff < 0)
            {
                item = default!;
                return false;  // Queue empty
            }
        }
    }

    private struct Cell<TValue>
    {
        public long Sequence;
        public TValue? Value;
    }
}
```

## Best Practices

### 1. Use Power-of-2 Capacity

```csharp
// ✅ Power of 2 enables fast modulo with bitwise AND
int capacity = 1024;  // 2^10
int index = position & (capacity - 1);  // Fast modulo

// ❌ Non-power-of-2 requires slow modulo
int index = position % capacity;
```

### 2. Pad to Prevent False Sharing

```csharp
[StructLayout(LayoutKind.Explicit, Size = 128)]
public class PaddedQueue<T>
{
    [FieldOffset(0)]
    private long head;  // Own cache line

    [FieldOffset(64)]
    private long tail;  // Different cache line
}
```

### 3. Profile Contention

```csharp
// ✅ Measure queue performance
var sw = Stopwatch.StartNew();
for (int i = 0; i < 1_000_000; i++)
{
    queue.TryEnqueue(i);
}
sw.Stop();

Console.WriteLine($"Throughput: {1_000_000 / sw.Elapsed.TotalSeconds:N0} ops/sec");
```

## Performance Characteristics

- **SPSC Queue**: 100M+ ops/sec, ~20ns latency
- **MPMC Queue**: 10-50M ops/sec, ~100ns latency  
- **Lock-Based**: 1-5M ops/sec, 500ns+ latency
- **Memory**: Bounded by capacity, no allocations

## When to Use

- High-frequency producer-consumer patterns
- Market data distribution pipelines
- Order routing systems
- Event streaming with multiple consumers

## When NOT to Use

- Unbounded queues (use channels with backpressure)
- Need priority ordering (use priority queue)
- Low-frequency scenarios (lock-based is simpler)

## Related Patterns

- [Lock-Free Data Structures](./lock-free-data-structures.md) — General lock-free patterns
- [Producer-Consumer Patterns](./producer-consumer-patterns.md) — Pipeline architectures
- [Cache Line Optimization](./cache-line-optimization.md) — False sharing prevention
- [Memory Pooling Strategies](./memory-pooling-strategies.md) — Combine with pooling

## References

- "Disruptor Pattern" by LMAX Exchange
- "Bounded MPMC Queue" by Dmitry Vyukov
- "Lock-Free Programming" by Herb Sutter
