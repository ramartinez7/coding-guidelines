# Thread Affinity

> Pin threads to specific CPU cores to reduce context switches—improve cache locality and eliminate NUMA effects.

## Problem

The OS scheduler moves threads between CPU cores to balance load, but each move invalidates CPU caches and introduces latency spikes. In low-latency systems, these context switches add hundreds of microseconds. Thread affinity pins threads to specific cores, ensuring hot data stays in L1/L2 cache and eliminating cross-NUMA memory access.

## Example

### ❌ Before (No Affinity)

```csharp
public class OrderProcessor
{
    public void Start()
    {
        var thread = new Thread(ProcessLoop);
        thread.Start();
        
        // OS scheduler migrates thread across cores
        // - Cache evictions on every migration
        // - NUMA penalties on remote memory access
        // - Unpredictable latencies
    }
}
```

### ✅ After (Pinned to Core)

```csharp
public class OrderProcessor
{
    private readonly int cpuCore;

    public OrderProcessor(int cpuCore)
    {
        this.cpuCore = cpuCore;
    }

    public void Start()
    {
        var thread = new Thread(ProcessLoop)
        {
            Priority = ThreadPriority.Highest,
            IsBackground = false
        };

        thread.Start();

        // Pin to specific CPU core
        SetThreadAffinity(thread, cpuCore);
    }

    [DllImport("kernel32.dll")]
    private static extern IntPtr SetThreadAffinityMask(IntPtr hThread, IntPtr dwThreadAffinityMask);

    [DllImport("kernel32.dll")]
    private static extern IntPtr GetCurrentThread();

    private static void SetThreadAffinity(Thread thread, int cpuCore)
    {
        if (RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
        {
            var mask = new IntPtr(1 << cpuCore);
            SetThreadAffinityMask(GetCurrentThread(), mask);
        }
        else if (RuntimeInformation.IsOSPlatform(OSPlatform.Linux))
        {
            // Linux: sched_setaffinity
            NativeLibrary.TryLoad("libc.so.6", out var handle);
            // Implementation details...
        }
    }
}
```

## Core Allocation Strategy

```csharp
public class CoreAllocator
{
    public static CoreAssignment AllocateCores()
    {
        var processorCount = Environment.ProcessorCount;
        var physicalCoreCount = GetPhysicalCoreCount();

        return new CoreAssignment
        {
            // Network I/O on dedicated core
            NetworkCore = 0,

            // Critical path on separate physical cores
            OrderProcessingCore = 2,
            RiskCheckCore = 4,
            MatchingEngineCore = 6,

            // Avoid hyperthreads for critical work
            // Cores 1, 3, 5, 7 are hyperthreads of 0, 2, 4, 6
        };
    }

    private static int GetPhysicalCoreCount()
    {
        // Detect physical vs logical cores
        return Environment.ProcessorCount / 2;  // Simplified
    }
}

public record CoreAssignment
{
    public int NetworkCore { get; init; }
    public int OrderProcessingCore { get; init; }
    public int RiskCheckCore { get; init; }
    public int MatchingEngineCore { get; init; }
}
```

## NUMA-Aware Allocation

```csharp
public class NumaAwareProcessor
{
    public void Start(int numaNode, int cpuCore)
    {
        var thread = new Thread(() =>
        {
            // Pin thread to NUMA node and CPU core
            SetNumaNode(numaNode);
            SetThreadAffinity(Thread.CurrentThread, cpuCore);

            // Allocate memory on same NUMA node
            AllocateNumaMemory(numaNode);

            ProcessLoop();
        });

        thread.Start();
    }

    [DllImport("kernel32.dll")]
    private static extern bool SetThreadIdealProcessor(IntPtr hThread, int dwIdealProcessor);

    private static void SetNumaNode(int node)
    {
        // Windows: SetThreadAffinityMask with NUMA awareness
        // Linux: numa_run_on_node
    }
}
```

## Best Practices

### 1. Reserve Cores for Critical Threads

```csharp
// ✅ Isolate critical cores from OS scheduler
// Boot parameter: isolcpus=2,4,6
// Or use cpuset on Linux

public static void ReserveCores()
{
    if (RuntimeInformation.IsOSPlatform(OSPlatform.Linux))
    {
        // echo "2,4,6" > /sys/fs/cgroup/cpuset/trading/cpuset.cpus
        // Move process to isolated cpuset
    }
}
```

### 2. Avoid Hyper-Threading for Critical Work

```csharp
// ✅ Use physical cores only
var physicalCores = new[] { 0, 2, 4, 6 };  // Even cores
var criticalThread = physicalCores[0];
var lessCriticalThread = physicalCores[1];

// ❌ Don't use hyperthreads
var hyperthreads = new[] { 1, 3, 5, 7 };  // Share resources with physical cores
```

### 3. Profile Cache Misses

```csharp
// ✅ Measure impact of affinity
public class AffinityBenchmark
{
    [Benchmark(Baseline = true)]
    public void NoAffinity()
    {
        // Let OS schedule across cores
        ProcessOrders();
    }

    [Benchmark]
    public void WithAffinity()
    {
        SetThreadAffinity(Thread.CurrentThread, 2);
        ProcessOrders();
    }
}
```

## Performance Impact

- **Cache hit rate**: 80% → 95% (pinned)
- **Latency variance**: p99 reduced by 50-70%
- **NUMA access**: Local memory 2-3x faster
- **Context switches**: Eliminated for critical threads

## When to Use

- Ultra-low latency requirements (<10μs)
- Dedicated hardware for trading
- Consistent performance critical
- High CPU cache utilization needed

## When NOT to Use

- Shared infrastructure
- Dynamic workloads
- Need for OS load balancing
- Development/testing environments

## Related Patterns

- [Latency-Sensitive Design](./latency-sensitive-design.md) — Low-latency techniques
- [Cache Line Optimization](./cache-line-optimization.md) — Cache efficiency
- [Busy-Wait vs Sleep](./busy-wait-vs-sleep.md) — Waiting strategies
- [Predictable Performance](./predictable-performance.md) — Consistency

## References

- "Systems Performance" by Brendan Gregg
- Intel® 64 Architecture Memory Ordering White Paper
- Linux NUMA documentation
