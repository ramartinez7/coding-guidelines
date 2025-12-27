# Memory Barriers and Synchronization

> Ensure correct ordering of memory operations across threads—use memory barriers to prevent instruction reordering and guarantee visibility of writes.

## Problem

Modern CPUs and compilers reorder instructions for performance. Without explicit synchronization, one thread's writes may not be visible to other threads, or operations may appear to execute out of order. This causes subtle, non-deterministic bugs in concurrent code—data corruption, lost updates, and race conditions that only appear under high load.

## Example

### ❌ Before (No Memory Barrier)

```csharp
public class UnsafeFlag
{
    private int value;
    private bool ready;
    
    // Writer thread
    public void Produce()
    {
        value = 42;      // Write 1
        ready = true;    // Write 2
        
        // CPU/compiler might reorder:
        // ready = true;  ← Executed first!
        // value = 42;    ← Executed second!
        
        // Reader sees ready=true but value=0!
    }
    
    // Reader thread
    public void Consume()
    {
        if (ready)  // May see ready=true
        {
            int v = value;  // But value might still be 0!
            // Race condition!
        }
    }
}
```

### ✅ After (With Memory Barrier)

```csharp
public class SafeFlag
{
    private int value;
    private volatile bool ready;  // Volatile provides barriers
    
    // Writer thread
    public void Produce()
    {
        value = 42;
        
        // Memory barrier: ensures value write completes
        Thread.MemoryBarrier();  // Or use volatile
        
        ready = true;  // This write happens AFTER value write
    }
    
    // Reader thread
    public void Consume()
    {
        if (ready)  // Volatile read provides acquire barrier
        {
            // Memory barrier: ensures we see all prior writes
            int v = value;  // Guaranteed to see value=42
        }
    }
}

// Or using Volatile explicitly:
public class SafeFlagExplicit
{
    private int value;
    private bool ready;
    
    public void Produce()
    {
        value = 42;
        Volatile.Write(ref ready, true);  // Release semantics
    }
    
    public void Consume()
    {
        if (Volatile.Read(ref ready))  // Acquire semantics
        {
            int v = value;  // Sees value=42
        }
    }
}
```

## Memory Barrier Types

```csharp
public static class MemoryBarriers
{
    // ✅ Full memory barrier
    public static void FullBarrier()
    {
        Thread.MemoryBarrier();  // Prevents all reordering
        
        // All reads/writes before this point complete
        // before any reads/writes after this point start
    }
    
    // ✅ Acquire barrier (load-load, load-store)
    public static int AcquireRead(ref int location)
    {
        int value = Volatile.Read(ref location);
        
        // Prevents reordering of this read with
        // subsequent reads and writes
        
        return value;
    }
    
    // ✅ Release barrier (load-store, store-store)
    public static void ReleaseWrite(ref int location, int value)
    {
        Volatile.Write(ref location, value);
        
        // Prevents reordering of prior reads/writes
        // with this write
    }
    
    // ✅ Interlocked operations include barriers
    public static void InterlockedBarrier(ref int location)
    {
        Interlocked.Increment(ref location);  // Full barrier
        Interlocked.CompareExchange(ref location, 1, 0);  // Full barrier
        Interlocked.Exchange(ref location, 1);  // Full barrier
    }
}
```

## Visual Explanation of Memory Barriers

```csharp
/// <summary>
/// Memory Barrier Visual Guide:
/// 
/// WITHOUT BARRIER:
/// Thread 1:        Thread 2:
/// x = 1            while (flag == false) {}
/// flag = true      print(x)
/// 
/// CPU might reorder:
/// flag = true      (sees flag=true)
/// x = 1            print(x) → prints 0! (race!)
/// 
/// WITH BARRIER:
/// Thread 1:                    Thread 2:
/// x = 1                        while (!Volatile.Read(ref flag)) {}
/// Volatile.Write(ref flag, true)   print(x)
/// ↑ Release barrier            ↑ Acquire barrier
/// 
/// Guarantees:
/// - Thread 1: x=1 completes before flag=true is visible
/// - Thread 2: flag=true seen means x=1 is also visible
/// </summary>
public class MemoryBarrierVisual
{
    private int x;
    private bool flag;
    
    public void ProducerWithBarrier()
    {
        x = 1;
        // ─────────────────────────
        // RELEASE BARRIER HERE
        // ─────────────────────────
        Volatile.Write(ref flag, true);
        
        // Guarantees x=1 visible before flag=true
    }
    
    public void ConsumerWithBarrier()
    {
        while (!Volatile.Read(ref flag)) { }
        // ─────────────────────────
        // ACQUIRE BARRIER HERE
        // ─────────────────────────
        
        Console.WriteLine(x);  // Always prints 1
    }
}
```

## Acquire-Release Semantics

```csharp
public class AcquireRelease
{
    private int data;
    private int flag;
    
    // ✅ Release: publish data
    public void Publish(int value)
    {
        data = value;
        
        // Release barrier: all prior writes visible
        Volatile.Write(ref flag, 1);
    }
    
    // ✅ Acquire: consume data
    public int Consume()
    {
        // Acquire barrier: see all prior writes
        int f = Volatile.Read(ref flag);
        
        if (f == 1)
        {
            return data;  // Guaranteed to see correct data
        }
        
        return -1;
    }
    
    // ✅ Ordering guarantees:
    // - Writes before Release stay before
    // - Reads after Acquire stay after
    // - Release-Acquire on same location = synchronizes
}
```

## Store Buffer and Cache Coherence

```csharp
/// <summary>
/// Store Buffer Visual:
/// 
/// CPU Core 1:              CPU Core 2:
/// [Register]               [Register]
///     ↓                        ↑
/// [Store Buffer]           [Load Buffer]
///     ↓                        ↑
/// [L1 Cache]               [L1 Cache]
///     ↓                        ↑
///      [L2/L3 Cache - Shared]
///              ↑↓
///            [Memory]
/// 
/// Without barriers:
/// - Core 1 writes stay in store buffer (invisible to Core 2)
/// - Core 2 reads from stale L1 cache
/// 
/// With barriers:
/// - Flush store buffer to L1
/// - Invalidate L1 cache on Core 2
/// - Core 2 sees updated value
/// </summary>
public class StoreBufferDemo
{
    private int x, y;
    private int r1, r2;
    
    // Thread 1
    public void Thread1()
    {
        x = 1;
        // Without barrier, stays in store buffer!
        r1 = y;  // Might read 0
    }
    
    // Thread 2
    public void Thread2()
    {
        y = 1;
        // Without barrier, stays in store buffer!
        r2 = x;  // Might read 0
    }
    
    // Possible outcome: r1=0, r2=0 (neither sees other's write!)
    
    // ✅ With barriers:
    public void Thread1Fixed()
    {
        x = 1;
        Thread.MemoryBarrier();  // Flush store buffer
        r1 = y;  // Guaranteed to see y after barrier
    }
}
```

## Happens-Before Relationship

```csharp
/// <summary>
/// Happens-Before Rules:
/// 
/// 1. Program Order: Statement A before B → A happens-before B
/// 2. Monitor Lock: Unlock happens-before subsequent lock
/// 3. Volatile Write: Happens-before subsequent volatile read
/// 4. Thread Start: Thread.Start() happens-before first statement
/// 5. Thread Join: Last statement happens-before Join() returns
/// 
/// Transitive: If A happens-before B, and B happens-before C,
///             then A happens-before C
/// </summary>
public class HappensBefore
{
    private int data;
    private volatile bool ready;
    
    public void Producer()
    {
        data = 42;         // A
        ready = true;      // B (volatile write)
        
        // A happens-before B (program order)
        // B happens-before any volatile read of ready
    }
    
    public void Consumer()
    {
        if (ready)         // C (volatile read)
        {
            int x = data;  // D
            
            // C happens-before D (program order)
            // B happens-before C (volatile semantics)
            // Therefore: A happens-before D (transitive)
            // x is guaranteed to be 42!
        }
    }
}
```

## Double-Checked Locking Pattern

```csharp
public class Singleton
{
    private static volatile Singleton? instance;
    private static readonly object lockObj = new();
    
    private Singleton() { }
    
    // ✅ Correct double-checked locking
    public static Singleton Instance
    {
        get
        {
            // First check (no lock, acquire semantics)
            if (instance == null)
            {
                lock (lockObj)  // Full barrier
                {
                    // Second check (inside lock)
                    if (instance == null)
                    {
                        instance = new Singleton();
                        // Volatile write: release semantics
                    }
                }
            }
            
            return instance;
        }
    }
    
    // ❌ Without volatile:
    // private static Singleton? instance;  // Wrong!
    // 
    // Problem: Constructor might not complete before
    // instance is visible to other threads!
}
```

## Market Data Publisher with Barriers

```csharp
public class MarketDataPublisher
{
    private decimal price;
    private int volume;
    private long sequence;
    private volatile bool updated;
    
    // ✅ Producer: publish market data
    public void PublishTick(decimal newPrice, int newVolume)
    {
        price = newPrice;
        volume = newVolume;
        
        // Increment sequence with full barrier
        Interlocked.Increment(ref sequence);
        
        // Release barrier: all writes visible
        Volatile.Write(ref updated, true);
    }
    
    // ✅ Consumer: read market data
    public bool TryReadTick(out decimal outPrice, out int outVolume, out long outSeq)
    {
        // Acquire barrier: see all prior writes
        if (Volatile.Read(ref updated))
        {
            outSeq = Interlocked.Read(ref sequence);
            outPrice = price;    // Guaranteed to see latest
            outVolume = volume;  // Guaranteed to see latest
            
            Volatile.Write(ref updated, false);
            return true;
        }
        
        outPrice = 0;
        outVolume = 0;
        outSeq = 0;
        return false;
    }
}
```

## Lock-Free Queue with Barriers

```csharp
public class LockFreeQueue<T>
{
    private class Node
    {
        public T Value;
        public Node? Next;
    }
    
    private Node? head;
    private Node? tail;
    
    public void Enqueue(T value)
    {
        var newNode = new Node { Value = value };
        
        while (true)
        {
            // Acquire: read current tail
            var currentTail = Volatile.Read(ref tail);
            
            if (currentTail == null)
            {
                // Try to set both head and tail
                if (Interlocked.CompareExchange(ref head, newNode, null) == null)
                {
                    Volatile.Write(ref tail, newNode);  // Release
                    return;
                }
            }
            else
            {
                // Try to link new node
                if (Interlocked.CompareExchange(ref currentTail.Next, newNode, null) == null)
                {
                    // CAS includes full barrier
                    Interlocked.CompareExchange(ref tail, newNode, currentTail);
                    return;
                }
            }
        }
    }
    
    public bool TryDequeue(out T value)
    {
        while (true)
        {
            // Acquire: read current head
            var currentHead = Volatile.Read(ref head);
            
            if (currentHead == null)
            {
                value = default!;
                return false;
            }
            
            var next = Volatile.Read(ref currentHead.Next);
            
            // CAS includes full barrier
            if (Interlocked.CompareExchange(ref head, next, currentHead) == currentHead)
            {
                value = currentHead.Value;
                return true;
            }
        }
    }
}
```

## Best Practices

### 1. Use Volatile for Flags

```csharp
// ✅ Volatile for shared flags
private volatile bool stopRequested;

public void RequestStop()
{
    stopRequested = true;  // Visible to all threads
}
```

### 2. Use Interlocked for Atomic Updates

```csharp
// ✅ Interlocked includes barriers
private int counter;

public void Increment()
{
    Interlocked.Increment(ref counter);  // Atomic + barriers
}
```

### 3. Use Thread.MemoryBarrier for Complex Scenarios

```csharp
// ✅ Full barrier when needed
public void ComplexUpdate()
{
    field1 = value1;
    field2 = value2;
    Thread.MemoryBarrier();  // Ensure both writes visible
    field3 = value3;
}
```

### 4. Document Synchronization Strategy

```csharp
/// <summary>
/// Synchronization: volatile 'ready' flag uses acquire-release semantics
/// to synchronize access to 'data' field without locks.
/// </summary>
private int data;
private volatile bool ready;
```

## Performance Characteristics

- **Memory Barrier**: ~10-20 cycles
- **Volatile Read**: 1-5 cycles extra vs normal read
- **Volatile Write**: 1-5 cycles extra vs normal write
- **Interlocked**: 20-100 cycles (includes barrier + atomic)

## When to Use

- Lock-free data structures
- Publishing data between threads
- Implementing flags and signals
- Ensuring visibility of writes
- Double-checked locking

## When NOT to Use

- Single-threaded code
- Already using locks (locks include barriers)
- Simple sequential code
- Low-contention scenarios (locks simpler)

## Related Patterns

- [Atomic Operations Patterns](./atomic-operations-patterns.md) — Interlocked
- [Volatile Explained](./volatile-explained.md) — Volatile keyword
- [Lock-Free Data Structures](./lock-free-data-structures.md) — Lock-free
- [Compare and Swap](./compare-and-swap-explained.md) — CAS operations

## References

- "The Art of Multiprocessor Programming" by Herlihy & Shavit
- "C# Memory Model" - ECMA-334 Specification
- "Memory Barriers: a Hardware View for Software Hackers" by McKenney
- "Intel® 64 and IA-32 Architectures Software Developer's Manual"
