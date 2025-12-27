# Disruptor Pattern

> Implement ultra-low latency inter-thread communication using ring buffers—achieve mechanical sympathy through cache-friendly, wait-free queuing.

## Problem

Traditional queues use locks, CAS loops, or channels that cause cache misses, context switches, and unpredictable latency. The Disruptor pattern, developed by LMAX Exchange for high-frequency trading, uses a pre-allocated ring buffer with sequence numbers to achieve single-digit microsecond latencies without locks or CAS loops in the hot path.

## Example

### ❌ Before (BlockingCollection)

```csharp
public class TraditionalQueue<T>
{
    private readonly BlockingCollection<T> queue = new();
    
    // Producer
    public void Publish(T item)
    {
        queue.Add(item);  // Lock, potential blocking, cache misses
    }
    
    // Consumer
    public void Process()
    {
        foreach (var item in queue.GetConsumingEnumerable())
        {
            ProcessItem(item);
        }
    }
    
    // Problems:
    // - Lock contention
    // - Context switches
    // - Unbounded allocations
    // - p99 latency: 100+µs
    
    private void ProcessItem(T item) { }
}
```

### ✅ After (Disruptor Ring Buffer)

```csharp
public class DisruptorRingBuffer<T> where T : class, new()
{
    private readonly T[] buffer;
    private readonly int bufferMask;
    private long writeSequence = -1;
    private long readSequence = -1;
    
    public DisruptorRingBuffer(int bufferSize)
    {
        // Buffer size must be power of 2
        if ((bufferSize & (bufferSize - 1)) != 0)
            throw new ArgumentException("Buffer size must be power of 2");
        
        this.buffer = new T[bufferSize];
        this.bufferMask = bufferSize - 1;
        
        // Preallocate all entries
        for (int i = 0; i < bufferSize; i++)
        {
            buffer[i] = new T();
        }
    }
    
    // Producer: claim next slot
    public long Next()
    {
        long current = Interlocked.Increment(ref writeSequence);
        
        // Wait if buffer full (spin wait)
        while (current - Volatile.Read(ref readSequence) > bufferMask)
        {
            Thread.SpinWait(1);  // Wait for consumer
        }
        
        return current;
    }
    
    // Producer: get entry for writing
    public T Get(long sequence)
    {
        return buffer[sequence & bufferMask];  // Fast modulo via mask
    }
    
    // Producer: publish entry
    public void Publish(long sequence)
    {
        // Entry already written, just make visible
        // No CAS needed: writeSequence already incremented
    }
    
    // Consumer: get next available sequence
    public bool TryGetNext(out long sequence)
    {
        long current = Volatile.Read(ref readSequence);
        long available = Volatile.Read(ref writeSequence);
        
        if (current >= available)
        {
            sequence = -1;
            return false;
        }
        
        sequence = current + 1;
        Interlocked.Exchange(ref readSequence, sequence);
        return true;
    }
    
    // ✅ Wait-free in steady state
    // ✅ No allocations after initialization
    // ✅ Cache-friendly sequential access
    // ✅ p99 latency: <1µs
}

// Usage
public class TradingSystem
{
    private readonly DisruptorRingBuffer<OrderEvent> disruptor = new(1024);
    
    // Producer thread
    public void PublishOrder(string symbol, decimal price)
    {
        long sequence = disruptor.Next();
        var order = disruptor.Get(sequence);
        
        order.Symbol = symbol;
        order.Price = price;
        order.Timestamp = DateTime.UtcNow.Ticks;
        
        disruptor.Publish(sequence);
    }
    
    // Consumer thread
    public void ProcessOrders()
    {
        while (true)
        {
            if (disruptor.TryGetNext(out long sequence))
            {
                var order = disruptor.Get(sequence);
                ProcessOrder(order);
            }
        }
    }
    
    private void ProcessOrder(OrderEvent order) { }
}

public class OrderEvent
{
    public string Symbol { get; set; } = "";
    public decimal Price { get; set; }
    public long Timestamp { get; set; }
}
```

## Sequence Barrier Pattern

```csharp
public class SequenceBarrier
{
    private readonly long[] sequences;
    
    public SequenceBarrier(int consumerCount)
    {
        // Pad to avoid false sharing
        this.sequences = new long[consumerCount * 16];
        
        for (int i = 0; i < consumerCount; i++)
        {
            sequences[i * 16] = -1;
        }
    }
    
    public void Set(int consumerId, long sequence)
    {
        Volatile.Write(ref sequences[consumerId * 16], sequence);
    }
    
    public long GetMinimum()
    {
        long min = long.MaxValue;
        
        for (int i = 0; i < sequences.Length; i += 16)
        {
            long seq = Volatile.Read(ref sequences[i]);
            if (seq < min)
                min = seq;
        }
        
        return min;
    }
}
```

## Multi-Producer Disruptor

```csharp
public class MultiProducerDisruptor<T> where T : class, new()
{
    private readonly T[] buffer;
    private readonly int bufferMask;
    private readonly int[] available;
    private long cursor = -1;
    private long readCursor = -1;
    
    public MultiProducerDisruptor(int bufferSize)
    {
        this.buffer = new T[bufferSize];
        this.bufferMask = bufferSize - 1;
        this.available = new int[bufferSize];
        
        for (int i = 0; i < bufferSize; i++)
        {
            buffer[i] = new T();
            available[i] = -1;
        }
    }
    
    // Producer: claim slot using CAS
    public long Next()
    {
        long current, next;
        
        do
        {
            current = Volatile.Read(ref cursor);
            next = current + 1;
            
            // Wait if buffer full
            long wrapPoint = next - buffer.Length;
            if (wrapPoint > Volatile.Read(ref readCursor))
            {
                Thread.SpinWait(1);
                continue;
            }
        }
        while (Interlocked.CompareExchange(ref cursor, next, current) != current);
        
        return next;
    }
    
    // Producer: publish slot
    public void Publish(long sequence)
    {
        // Mark sequence as available
        int index = (int)(sequence & bufferMask);
        Volatile.Write(ref available[index], 1);
    }
    
    // Consumer: wait for next available
    public long WaitFor(long sequence)
    {
        while (true)
        {
            int index = (int)(sequence & bufferMask);
            
            if (Volatile.Read(ref available[index]) == 1)
            {
                // Mark as consumed
                Volatile.Write(ref available[index], -1);
                return sequence;
            }
            
            Thread.SpinWait(1);
        }
    }
    
    public T Get(long sequence)
    {
        return buffer[sequence & bufferMask];
    }
}
```

## Event Processor Pattern

```csharp
public interface IEventHandler<T>
{
    void OnEvent(T data, long sequence, bool endOfBatch);
}

public class EventProcessor<T> where T : class, new()
{
    private readonly DisruptorRingBuffer<T> ringBuffer;
    private readonly IEventHandler<T> handler;
    private readonly Thread thread;
    private volatile bool running;
    
    public EventProcessor(DisruptorRingBuffer<T> ringBuffer, IEventHandler<T> handler)
    {
        this.ringBuffer = ringBuffer;
        this.handler = handler;
        this.thread = new Thread(Run)
        {
            Name = "EventProcessor",
            IsBackground = false
        };
    }
    
    public void Start()
    {
        running = true;
        thread.Start();
    }
    
    public void Stop()
    {
        running = false;
        thread.Join();
    }
    
    private void Run()
    {
        long nextSequence = 0;
        
        while (running)
        {
            if (ringBuffer.TryGetNext(out long sequence))
            {
                var data = ringBuffer.Get(sequence);
                bool endOfBatch = sequence == ringBuffer.GetCursor();
                
                handler.OnEvent(data, sequence, endOfBatch);
                nextSequence = sequence + 1;
            }
            else
            {
                // No data available, spin
                Thread.SpinWait(1);
            }
        }
    }
}

public static class RingBufferExtensions
{
    public static long GetCursor<T>(this DisruptorRingBuffer<T> buffer) where T : class, new()
    {
        // Implementation to get current cursor
        return 0;  // Placeholder
    }
}
```

## Batching Support

```csharp
public class BatchingDisruptor<T> where T : class, new()
{
    private readonly DisruptorRingBuffer<T> ringBuffer;
    private readonly int batchSize;
    
    public BatchingDisruptor(int bufferSize, int batchSize)
    {
        this.ringBuffer = new DisruptorRingBuffer<T>(bufferSize);
        this.batchSize = batchSize;
    }
    
    // Publish batch
    public void PublishBatch(T[] items)
    {
        long endSequence = ringBuffer.Next();
        long startSequence = endSequence - items.Length + 1;
        
        for (int i = 0; i < items.Length; i++)
        {
            long seq = startSequence + i;
            var entry = ringBuffer.Get(seq);
            CopyFrom(entry, items[i]);
            ringBuffer.Publish(seq);
        }
    }
    
    // Process batch
    public void ProcessBatch(Action<T[], int> batchHandler)
    {
        var batch = new T[batchSize];
        int batchCount = 0;
        
        while (true)
        {
            if (ringBuffer.TryGetNext(out long sequence))
            {
                batch[batchCount++] = ringBuffer.Get(sequence);
                
                if (batchCount >= batchSize)
                {
                    batchHandler(batch, batchCount);
                    batchCount = 0;
                }
            }
            else if (batchCount > 0)
            {
                // Process partial batch
                batchHandler(batch, batchCount);
                batchCount = 0;
            }
        }
    }
    
    private void CopyFrom(T dest, T source)
    {
        // Copy fields from source to dest
    }
}
```

## Wait Strategy Patterns

```csharp
public interface IWaitStrategy
{
    void WaitFor(long sequence, Func<long> getAvailableSequence);
    void SignalAllWhenBlocking();
}

// ✅ Busy spin (lowest latency, highest CPU)
public class BusySpinWaitStrategy : IWaitStrategy
{
    public void WaitFor(long sequence, Func<long> getAvailableSequence)
    {
        while (getAvailableSequence() < sequence)
        {
            // Busy spin
        }
    }
    
    public void SignalAllWhenBlocking() { }
}

// ✅ Yielding (moderate latency, moderate CPU)
public class YieldingWaitStrategy : IWaitStrategy
{
    private int spinCount;
    
    public void WaitFor(long sequence, Func<long> getAvailableSequence)
    {
        int spin = 0;
        
        while (getAvailableSequence() < sequence)
        {
            if (++spin > 100)
            {
                Thread.Yield();
                spin = 0;
            }
        }
    }
    
    public void SignalAllWhenBlocking() { }
}

// ✅ Sleeping (higher latency, low CPU)
public class SleepingWaitStrategy : IWaitStrategy
{
    public void WaitFor(long sequence, Func<long> getAvailableSequence)
    {
        int spin = 0;
        
        while (getAvailableSequence() < sequence)
        {
            if (++spin > 100)
            {
                Thread.Sleep(0);  // Yield to other threads
                spin = 0;
            }
            else if (spin > 200)
            {
                Thread.Sleep(1);  // Sleep 1ms
            }
        }
    }
    
    public void SignalAllWhenBlocking() { }
}
```

## Market Data Pipeline

```csharp
public class MarketDataPipeline
{
    private readonly DisruptorRingBuffer<MarketTick> ringBuffer = new(8192);
    
    // Stage 1: Network receiver (producer)
    public void ReceiveTick(byte[] data)
    {
        long sequence = ringBuffer.Next();
        var tick = ringBuffer.Get(sequence);
        
        ParseTick(data, tick);
        ringBuffer.Publish(sequence);
    }
    
    // Stage 2: Validator (consumer + producer)
    public void ValidatorLoop()
    {
        while (true)
        {
            if (ringBuffer.TryGetNext(out long sequence))
            {
                var tick = ringBuffer.Get(sequence);
                
                if (IsValid(tick))
                {
                    // Forward to next stage
                    PublishToJournal(tick);
                }
            }
        }
    }
    
    // Stage 3: Journal writer (consumer)
    public void JournalLoop()
    {
        while (true)
        {
            if (ringBuffer.TryGetNext(out long sequence))
            {
                var tick = ringBuffer.Get(sequence);
                WriteToJournal(tick);
            }
        }
    }
    
    private void ParseTick(byte[] data, MarketTick tick) { }
    private bool IsValid(MarketTick tick) => true;
    private void PublishToJournal(MarketTick tick) { }
    private void WriteToJournal(MarketTick tick) { }
}

public class MarketTick
{
    public string Symbol { get; set; } = "";
    public decimal Price { get; set; }
    public int Volume { get; set; }
    public long Timestamp { get; set; }
}
```

## Best Practices

### 1. Power-of-2 Buffer Size

```csharp
// ✅ Use power of 2 for fast modulo
int bufferSize = 1024;  // 2^10
int index = sequence & (bufferSize - 1);  // Fast modulo
```

### 2. Preallocate Everything

```csharp
// ✅ Allocate all entries at creation
for (int i = 0; i < buffer.Length; i++)
{
    buffer[i] = new T();
}
```

### 3. Pad to Avoid False Sharing

```csharp
// ✅ Pad sequence numbers
[StructLayout(LayoutKind.Explicit, Size = 128)]
public struct PaddedLong
{
    [FieldOffset(64)]
    public long Value;
}
```

### 4. Choose Right Wait Strategy

```csharp
// ✅ Ultra-low latency: BusySpinWaitStrategy
// ✅ Moderate latency: YieldingWaitStrategy
// ✅ Low CPU usage: SleepingWaitStrategy
```

## Performance Characteristics

- **Latency**: 50ns - 1µs (vs 100µs+ for queues)
- **Throughput**: 100M+ ops/sec
- **Memory**: Fixed, zero GC pressure
- **CPU**: High (busy spinning)

## When to Use

- Ultra-low latency requirements (<10µs)
- High-throughput event processing
- Trading systems, real-time analytics
- Single or few producers/consumers
- Predictable load patterns

## When NOT to Use

- Many producers/consumers
- Unpredictable traffic patterns
- CPU constrained systems
- Occasional communications

## Related Patterns

- [Lock-Free Queues](./lock-free-queues.md) — Alternative queuing
- [Producer-Consumer Patterns](./producer-consumer-patterns.md) — Pipeline design
- [Cache Line Optimization](./cache-line-optimization.md) — Cache friendly
- [Wait-Free Algorithms](./wait-free-algorithms.md) — Wait-free guarantees

## References

- "Disruptor: High Performance Inter-Thread Messaging" - LMAX
- "Mechanical Sympathy" blog by Martin Thompson
- "Disruptor .NET" - GitHub implementation
- "The LMAX Architecture" by Martin Fowler
