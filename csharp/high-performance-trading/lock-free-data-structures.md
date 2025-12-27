# Lock-Free Data Structures

> Implement concurrent data structures without locks using atomic operations and compare-and-swap—eliminate contention and achieve wait-free guarantees for critical paths.

## Problem

Traditional lock-based data structures cause contention under high concurrency. When multiple threads compete for a lock, some must wait, leading to unpredictable latencies and potential deadlocks. In high-frequency trading systems where microseconds matter, lock contention is unacceptable.

## Example

### ❌ Before (Lock-Based Stack)

```csharp
public class LockedStack<T>
{
    private readonly object lockObj = new();
    private Node? head;

    public void Push(T value)
    {
        lock (lockObj)  // Threads block here
        {
            var newNode = new Node(value, head);
            head = newNode;
        }
    }

    public bool TryPop(out T value)
    {
        lock (lockObj)  // Contention on every operation
        {
            if (head == null)
            {
                value = default!;
                return false;
            }

            value = head.Value;
            head = head.Next;
            return true;
        }
    }

    private record Node(T Value, Node? Next);
}

// Problems:
// - Lock contention under high load
// - Potential deadlocks with nested locks
// - Unpredictable latencies
// - Priority inversion possible
```

### ✅ After (Lock-Free Stack)

```csharp
public class LockFreeStack<T>
{
    private Node? head;

    public void Push(T value)
    {
        var newNode = new Node(value);

        while (true)
        {
            // Read current head
            var currentHead = Volatile.Read(ref head);
            newNode.Next = currentHead;

            // Try to swap: only succeeds if head hasn't changed
            if (Interlocked.CompareExchange(ref head, newNode, currentHead) == currentHead)
            {
                return;  // Success!
            }

            // If failed, someone else modified head, retry
        }
    }

    public bool TryPop(out T value)
    {
        while (true)
        {
            var currentHead = Volatile.Read(ref head);

            if (currentHead == null)
            {
                value = default!;
                return false;
            }

            // Try to update head to next node
            if (Interlocked.CompareExchange(ref head, currentHead.Next, currentHead) == currentHead)
            {
                value = currentHead.Value;
                return true;
            }

            // Retry if someone else modified head
        }
    }

    private class Node
    {
        public T Value;
        public Node? Next;

        public Node(T value)
        {
            this.Value = value;
        }
    }
}
```

## Lock-Free Counter

```csharp
public sealed class LockFreeCounter
{
    private long count;

    public void Increment()
    {
        Interlocked.Increment(ref count);
    }

    public void Add(long value)
    {
        Interlocked.Add(ref count, value);
    }

    public long GetValue()
    {
        return Interlocked.Read(ref count);
    }

    // ✅ Wait-free: always completes in bounded time
}
```

## Lock-Free Queue (Simple FIFO)

```csharp
public class LockFreeQueue<T>
{
    private Node? head;
    private Node? tail;

    public void Enqueue(T value)
    {
        var newNode = new Node(value);

        while (true)
        {
            var currentTail = Volatile.Read(ref tail);

            if (currentTail == null)
            {
                // Empty queue
                if (Interlocked.CompareExchange(ref head, newNode, null) == null)
                {
                    Interlocked.CompareExchange(ref tail, newNode, null);
                    return;
                }
            }
            else
            {
                // Try to link new node to tail
                if (Interlocked.CompareExchange(ref currentTail.Next, newNode, null) == null)
                {
                    // Try to swing tail to new node
                    Interlocked.CompareExchange(ref tail, newNode, currentTail);
                    return;
                }
                else
                {
                    // Help other threads by updating tail
                    Interlocked.CompareExchange(ref tail, currentTail.Next, currentTail);
                }
            }
        }
    }

    public bool TryDequeue(out T value)
    {
        while (true)
        {
            var currentHead = Volatile.Read(ref head);

            if (currentHead == null)
            {
                value = default!;
                return false;
            }

            var next = Volatile.Read(ref currentHead.Next);

            if (Interlocked.CompareExchange(ref head, next, currentHead) == currentHead)
            {
                value = currentHead.Value;

                if (next == null)
                {
                    Interlocked.CompareExchange(ref tail, null, currentHead);
                }

                return true;
            }
        }
    }

    private class Node
    {
        public T Value;
        public Node? Next;

        public Node(T value)
        {
            this.Value = value;
        }
    }
}
```

## ABA Problem Solution

```csharp
// ❌ ABA Problem: head changes from A → B → A, CAS succeeds incorrectly
public class NaiveLockFreeStack<T>
{
    private Node? head;

    // If head is A, thread1 reads A
    // Thread2 pops A, pops B, pushes A back (now recycled)
    // Thread1's CAS succeeds but Next pointer may be corrupted!
}

// ✅ Solution: Use version counter or hazard pointers
public class ABASafeLockFreeStack<T>
{
    private VersionedNode head;

    public void Push(T value)
    {
        var newNode = new Node(value);

        while (true)
        {
            var currentHead = Volatile.Read(ref head);
            newNode.Next = currentHead.Node;

            var newHead = new VersionedNode(newNode, currentHead.Version + 1);

            if (Interlocked.CompareExchange(ref head, newHead, currentHead).Equals(currentHead))
            {
                return;
            }
        }
    }

    private record struct VersionedNode(Node? Node, long Version);

    private class Node
    {
        public T Value;
        public Node? Next;

        public Node(T value)
        {
            this.Value = value;
        }
    }
}
```

## Atomic Operations Reference

```csharp
public static class AtomicOperations
{
    // ✅ Compare-And-Swap (CAS)
    public static T CompareExchange<T>(ref T location, T value, T comparand)
        where T : class?
    {
        return Interlocked.CompareExchange(ref location, value, comparand);
    }

    // ✅ Atomic read
    public static T Read<T>(ref T location) where T : class?
    {
        return Volatile.Read(ref location);
    }

    // ✅ Atomic write
    public static void Write<T>(ref T location, T value) where T : class?
    {
        Volatile.Write(ref location, value);
    }

    // ✅ Atomic increment
    public static long Increment(ref long location)
    {
        return Interlocked.Increment(ref location);
    }

    // ✅ Atomic exchange
    public static T Exchange<T>(ref T location, T value) where T : class?
    {
        return Interlocked.Exchange(ref location, value);
    }
}
```

## Best Practices

### 1. Use Volatile for Visibility

```csharp
// ❌ May see stale values
private Node? head;

// ✅ Ensures visibility across threads
private Node? head;
var currentHead = Volatile.Read(ref head);
```

### 2. Avoid ABA Problem

```csharp
// ✅ Use version counters, hazard pointers, or managed memory
private record struct VersionedPointer<T>(T? Value, long Version);
```

### 3. Bounded Retry Loops

```csharp
// ❌ Infinite loop under extreme contention
while (true)
{
    if (TryOperation())
        return;
}

// ✅ Fallback after too many retries
for (int retry = 0; retry < MaxRetries; retry++)
{
    if (TryOperation())
        return;

    if (retry > 10)
        Thread.SpinWait(1 << (retry - 10));  // Exponential backoff
}

throw new TimeoutException("Operation failed after max retries");
```

### 4. Memory Ordering

```csharp
// ✅ Use memory barriers when needed
Interlocked.MemoryBarrier();  // Full fence

// ✅ Or use Volatile for acquire/release semantics
Volatile.Read(ref value);   // Acquire
Volatile.Write(ref value, newValue);  // Release
```

## Performance Characteristics

- **Push/Pop**: O(1) expected, wait-free under low contention
- **No locks**: Eliminates contention, deadlocks, priority inversion
- **Cache-friendly**: Compare-and-swap uses single cache line
- **Contention**: Performance degrades under extreme contention (use backoff)

## When to Use

- High-frequency operations (>100k ops/sec per thread)
- Low-latency requirements (<1 microsecond)
- Many concurrent readers/writers
- Cannot tolerate lock contention

## When NOT to Use

- Complex data structures requiring multiple coordinated updates
- Low contention scenarios (locks may be simpler)
- Need for fairness guarantees (locks with queues are better)

## Related Patterns

- [Wait-Free Algorithms](./wait-free-algorithms.md) — Stronger guarantees
- [Lock-Free Queues](./lock-free-queues.md) — Specialized FIFO queues
- [Memory Pooling Strategies](./memory-pooling-strategies.md) — Reduce allocations
- [Thread Safety Patterns](../patterns/thread-safety-patterns.md) — General thread safety

## References

- "The Art of Multiprocessor Programming" by Herlihy & Shavit
- "C# in Depth" by Jon Skeet (Chapter on concurrency)
- Intel® 64 and IA-32 Architectures Software Developer's Manual (Memory Ordering)
