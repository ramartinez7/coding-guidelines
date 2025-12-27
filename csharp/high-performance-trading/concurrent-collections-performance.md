# Concurrent Collections Performance

> Select the right concurrent collection for your use case—understand performance characteristics of ConcurrentQueue, ConcurrentBag, and channels.

## Problem

.NET provides several concurrent collections, each optimized for different access patterns. Choosing the wrong collection can reduce throughput by 10-100x. ConcurrentQueue excels at FIFO, ConcurrentBag at work-stealing, and Channels at async operations. Understanding their implementations and tradeoffs is critical for high-performance systems.

## Collections Comparison

### ConcurrentQueue<T>

```csharp
// ✅ Best for FIFO producer-consumer
public class FifoProcessor
{
    private readonly ConcurrentQueue<Order> queue = new();

    public void Enqueue(Order order)
    {
        queue.Enqueue(order);  // Lock-free, O(1)
    }

    public bool TryDequeue(out Order order)
    {
        return queue.TryDequeue(out order);  // Lock-free, O(1)
    }
}

// Characteristics:
// - Lock-free using CAS operations
// - FIFO ordering guaranteed
// - Good for single/multiple producers and consumers
```

### ConcurrentBag<T>

```csharp
// ✅ Best for work-stealing scenarios
public class ParallelWorkProcessor
{
    private readonly ConcurrentBag<WorkItem> bag = new();

    public void AddWork(WorkItem item)
    {
        bag.Add(item);  // Thread-local, very fast
    }

    public void ProcessWork()
    {
        Parallel.ForEach(Enumerable.Range(0, Environment.ProcessorCount), _ =>
        {
            while (bag.TryTake(out var item))
            {
                ProcessItem(item);
            }
        });
    }
}

// Characteristics:
// - Thread-local stacks for each thread
// - No FIFO/LIFO guarantee
// - Excellent for same thread adds/removes
```

### Channel<T>

```csharp
// ✅ Best for async producer-consumer
public class AsyncPipeline
{
    private readonly Channel<Tick> channel;

    public AsyncPipeline(int capacity = 1000)
    {
        channel = Channel.CreateBounded<Tick>(new BoundedChannelOptions(capacity)
        {
            FullMode = BoundedChannelFullMode.DropOldest,
            SingleReader = true,
            SingleWriter = true
        });
    }

    public async ValueTask ProduceAsync(Tick tick, CancellationToken token)
    {
        await channel.Writer.WriteAsync(tick, token);
    }

    public async ValueTask ConsumeAsync(CancellationToken token)
    {
        await foreach (var tick in channel.Reader.ReadAllAsync(token))
        {
            ProcessTick(tick);
        }
    }
}

// Characteristics:
// - Async/await friendly
// - Backpressure support
// - FIFO ordering
```

## When to Use Each

**ConcurrentQueue**: FIFO producer-consumer with ordering guarantees  
**ConcurrentBag**: Work-stealing patterns, Parallel.ForEach workloads  
**ConcurrentStack**: LIFO patterns, simpler than queue  
**ConcurrentDictionary**: Shared key-value store, read-heavy workloads  
**Channel**: Async operations, backpressure support, pipelines  
**Custom Lock-Free**: Ultra-low latency, maximum performance

## Related Patterns

- [Lock-Free Queues](./lock-free-queues.md) — Custom lock-free implementations
- [Producer-Consumer Patterns](./producer-consumer-patterns.md) — Pipeline architectures
- [Thread Safety Patterns](../patterns/thread-safety-patterns.md) — General concurrency
- [Lock-Free Data Structures](./lock-free-data-structures.md) — Lock-free patterns
