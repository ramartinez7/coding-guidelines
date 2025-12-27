# Producer-Consumer Patterns

> Design efficient producer-consumer pipelines for market data—separate concerns and maximize throughput with proper buffering.

## Problem

Market data systems must process millions of messages per second while maintaining low latency. Poorly designed producer-consumer patterns create bottlenecks, buffer overflows, and unpredictable latencies. The key is choosing the right queue structure, buffering strategy, and backpressure handling for your workload.

## Example

### ❌ Before (Blocking Queue with No Backpressure)

```csharp
public class MarketDataPipeline
{
    private readonly BlockingCollection<Tick> queue = new();

    // Producer: network thread
    public void OnTickReceived(Tick tick)
    {
        queue.Add(tick);  // Blocks if queue full!
        // Network thread stalls, causing message loss
    }

    // Consumer: processing thread
    public void ProcessLoop()
    {
        foreach (var tick in queue.GetConsumingEnumerable())
        {
            ProcessTick(tick);  // Slow processing blocks everything
        }
    }
}
```

### ✅ After (Lock-Free Queue with Backpressure)

```csharp
public class MarketDataPipeline
{
    private readonly SPSCQueue<Tick> queue;
    private readonly ILogger logger;
    private long droppedTicks;

    public MarketDataPipeline(int capacity = 65536)
    {
        queue = new SPSCQueue<Tick>(capacity);
    }

    // Producer: network thread (never blocks)
    public void OnTickReceived(Tick tick)
    {
        if (!queue.TryEnqueue(tick))
        {
            // Queue full: apply backpressure
            Interlocked.Increment(ref droppedTicks);
            logger.LogWarning("Dropped tick: queue full");
        }
    }

    // Consumer: dedicated processing thread
    public void ProcessLoop(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            if (queue.TryDequeue(out var tick))
            {
                ProcessTick(tick);
            }
            else
            {
                Thread.SpinWait(10);  // Busy-wait when empty
            }
        }
    }
}
```

## Multi-Stage Pipeline

```csharp
public class MultiStagePipeline
{
    private readonly SPSCQueue<RawMessage> parseQueue;
    private readonly SPSCQueue<ParsedTick> processQueue;
    private readonly SPSCQueue<AggregatedData> outputQueue;

    // Stage 1: Parse raw messages
    public void ParserStage(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            if (parseQueue.TryDequeue(out var raw))
            {
                var parsed = Parse(raw);
                while (!processQueue.TryEnqueue(parsed))
                {
                    Thread.SpinWait(1);  // Spin until space available
                }
            }
            else
            {
                Thread.SpinWait(10);
            }
        }
    }

    // Stage 2: Process ticks
    public void ProcessorStage(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            if (processQueue.TryDequeue(out var tick))
            {
                var aggregated = Aggregate(tick);
                while (!outputQueue.TryEnqueue(aggregated))
                {
                    Thread.SpinWait(1);
                }
            }
            else
            {
                Thread.SpinWait(10);
            }
        }
    }

    // Stage 3: Output results
    public void OutputStage(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            if (outputQueue.TryDequeue(out var data))
            {
                WriteOutput(data);
            }
            else
            {
                Thread.SpinWait(10);
            }
        }
    }
}
```

## Batching Pattern

```csharp
public class BatchingPipeline
{
    private readonly SPSCQueue<Tick> queue;
    private readonly int batchSize = 100;

    public void ProcessLoop(CancellationToken token)
    {
        var batch = new Tick[batchSize];

        while (!token.IsCancellationRequested)
        {
            int count = 0;

            // Collect batch
            while (count < batchSize && queue.TryDequeue(out var tick))
            {
                batch[count++] = tick;
            }

            if (count > 0)
            {
                // Process entire batch
                ProcessBatch(batch.AsSpan(0, count));
            }
            else
            {
                Thread.SpinWait(10);
            }
        }
    }

    private void ProcessBatch(Span<Tick> ticks)
    {
        // Batch processing is more efficient
        // - Amortizes overhead
        // - Better cache usage
        // - Vectorization opportunities
    }
}
```

## Fan-Out Pattern

```csharp
public class FanOutPipeline
{
    private readonly SPSCQueue<Tick> input;
    private readonly MPMCQueue<Tick>[] outputs;

    public FanOutPipeline(int consumerCount, int queueSize = 4096)
    {
        input = new SPSCQueue<Tick>(queueSize);
        outputs = new MPMCQueue<Tick>[consumerCount];

        for (int i = 0; i < consumerCount; i++)
        {
            outputs[i] = new MPMCQueue<Tick>(queueSize);
        }
    }

    // Producer
    public void Distribute(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            if (input.TryDequeue(out var tick))
            {
                // Fan out to all consumers
                foreach (var queue in outputs)
                {
                    while (!queue.TryEnqueue(tick))
                    {
                        Thread.SpinWait(1);
                    }
                }
            }
            else
            {
                Thread.SpinWait(10);
            }
        }
    }

    // Each consumer gets all ticks
    public void Consumer(int consumerId, CancellationToken token)
    {
        var queue = outputs[consumerId];

        while (!token.IsCancellationRequested)
        {
            if (queue.TryDequeue(out var tick))
            {
                ProcessTick(tick);
            }
            else
            {
                Thread.SpinWait(10);
            }
        }
    }
}
```

## Best Practices

### 1. Size Queues Appropriately

```csharp
// ✅ Based on message rate and processing time
var messagesPerSecond = 1_000_000;
var processingTimeMs = 10;
var bufferMultiplier = 4;  // Safety margin

var queueSize = (messagesPerSecond * processingTimeMs / 1000) * bufferMultiplier;
queueSize = NextPowerOf2(queueSize);  // Round up to power of 2
```

### 2. Use Dedicated Threads

```csharp
// ✅ Pin to specific CPU cores
var thread = new Thread(ProcessLoop)
{
    Priority = ThreadPriority.Highest,
    IsBackground = false
};

if (RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
{
    SetThreadAffinityMask(thread, cpuCore);
}

thread.Start();
```

### 3. Monitor Queue Depth

```csharp
// ✅ Track queue utilization
public class MonitoredQueue<T>
{
    private SPSCQueue<T> queue;
    private long maxDepth;

    public long GetDepth()
    {
        long depth = tail - head;
        
        long currentMax = Volatile.Read(ref maxDepth);
        if (depth > currentMax)
        {
            Interlocked.CompareExchange(ref maxDepth, depth, currentMax);
        }

        return depth;
    }
}
```

### 4. Handle Backpressure

```csharp
// ✅ Drop or throttle when overloaded
if (!queue.TryEnqueue(item))
{
    // Option 1: Drop
    Interlocked.Increment(ref droppedCount);

    // Option 2: Throttle producer
    Thread.Sleep(1);
    queue.TryEnqueue(item);

    // Option 3: Apply selective dropping
    if (item.Priority == Priority.Low)
        return;  // Drop low-priority items
}
```

## Performance Characteristics

- **SPSC Pipeline**: 100M+ ops/sec per stage
- **Multi-stage**: Throughput = slowest stage
- **Fan-out**: Throughput / consumer count
- **Batching**: 2-5x improvement for batch processing

## When to Use

- Market data distribution
- Order routing pipelines
- Real-time analytics
- Event processing systems

## Related Patterns

- [Lock-Free Queues](./lock-free-queues.md) — Queue implementations
- [Thread Affinity](./thread-affinity.md) — CPU pinning
- [Hot Path Optimization](./hot-path-optimization.md) — Optimize stages
- [Latency-Sensitive Design](./latency-sensitive-design.md) — Low-latency techniques
