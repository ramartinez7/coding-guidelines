# Compare-and-Swap (CAS) Explained

> Master the fundamental building block of lock-free programming—understand CAS operations, retry loops, ABA problem, and how to build lock-free algorithms.

## Problem

Lock-free algorithms need a way to atomically update shared state only if it hasn't changed since being read. Simple read-modify-write sequences aren't atomic and cause race conditions. Compare-and-Swap (CAS) solves this by atomically checking if a value matches an expected value and only updating if it does—enabling lock-free programming.

## Example

### ❌ Before (Race Condition)

```csharp
public class RacyCounter
{
    private int count;
    
    public void Increment()
    {
        int temp = count;      // Read
        temp = temp + 1;       // Modify
        count = temp;          // Write
        
        // Problem: Another thread can modify count between read and write!
        // Thread 1: reads 0
        // Thread 2: reads 0, writes 1
        // Thread 1: writes 1
        // Lost update! Should be 2, but is 1
    }
}
```

### ✅ After (CAS-Based)

```csharp
public class CASCounter
{
    private int count;
    
    public void Increment()
    {
        int current, next;
        
        do
        {
            current = Volatile.Read(ref count);  // Read current value
            next = current + 1;                  // Compute new value
            
            // Try to update: only succeeds if count still equals current
            // Returns OLD value
        }
        while (Interlocked.CompareExchange(ref count, next, current) != current);
        
        // If CAS fails, another thread changed count → retry
        // If CAS succeeds, count is atomically updated
    }
}
```

## CAS Operation Explained

```csharp
/// <summary>
/// Compare-and-Swap (CAS) Visual Explanation:
/// 
/// CompareExchange(ref location, newValue, expected)
/// 
/// Atomic operation:
/// 1. Read location
/// 2. If location == expected:
///       location = newValue
///       return expected (old value)
///    Else:
///       return current value (unchanged)
/// 
/// All 3 steps happen ATOMICALLY (single instruction: CMPXCHG)
/// </summary>
public static class CASExplained
{
    // ✅ CAS in pseudocode
    public static T CompareAndSwap<T>(ref T location, T newValue, T expected)
    {
        // This is atomic in hardware:
        T oldValue = location;
        
        if (oldValue == expected)
        {
            location = newValue;
        }
        
        return oldValue;
    }
    
    // ✅ Usage pattern
    public static void CASPattern<T>(ref T location, Func<T, T> transform)
    {
        T current, next;
        
        do
        {
            current = location;           // Read
            next = transform(current);    // Compute new value
        }
        while (Interlocked.CompareExchange(ref location, next, current) != current);
        // Retry if CAS fails (someone else modified location)
    }
}
```

## CAS Retry Loop Patterns

```csharp
public static class CASRetryLoops
{
    // ✅ Basic retry loop
    public static void BasicCASLoop(ref long location, long value)
    {
        long current;
        
        do
        {
            current = Interlocked.Read(ref location);
        }
        while (Interlocked.CompareExchange(ref location, value, current) != current);
    }
    
    // ✅ Conditional update
    public static bool ConditionalUpdate(ref long location, Func<long, bool> condition, Func<long, long> update)
    {
        long current, next;
        
        do
        {
            current = Interlocked.Read(ref location);
            
            if (!condition(current))
                return false;  // Don't update
            
            next = update(current);
        }
        while (Interlocked.CompareExchange(ref location, next, current) != current);
        
        return true;
    }
    
    // ✅ Retry with exponential backoff
    public static void CASWithBackoff(ref long location, long value)
    {
        long current;
        int retries = 0;
        
        while (true)
        {
            current = Interlocked.Read(ref location);
            
            if (Interlocked.CompareExchange(ref location, value, current) == current)
                break;  // Success!
            
            // Exponential backoff on contention
            if (retries > 10)
            {
                Thread.SpinWait(1 << Math.Min(retries - 10, 10));
            }
            
            retries++;
            
            if (retries > 1000)
            {
                Thread.Yield();  // Give up CPU
            }
        }
    }
    
    // ✅ Bounded retry with timeout
    public static bool CASWithTimeout(ref long location, long value, int maxRetries)
    {
        long current;
        
        for (int i = 0; i < maxRetries; i++)
        {
            current = Interlocked.Read(ref location);
            
            if (Interlocked.CompareExchange(ref location, value, current) == current)
                return true;
            
            if (i > 10)
                Thread.SpinWait(1 << Math.Min(i - 10, 10));
        }
        
        return false;  // Failed after max retries
    }
}
```

## ABA Problem

```csharp
/// <summary>
/// ABA Problem Visual:
/// 
/// Thread 1:                    Thread 2:
/// 1. Read: head = A            
/// 2. Next = B
/// 3. (paused)                  4. Pop A (head = B)
///                              5. Pop B (head = null)
///                              6. Push A (head = A, but DIFFERENT A!)
/// 7. CAS succeeds (head == A)  
/// 8. head.Next might be invalid!
/// 
/// Problem: head changed A → B → A
/// CAS sees "A" and succeeds, but it's a DIFFERENT A!
/// </summary>
public class ABAExample
{
    private class Node
    {
        public int Value;
        public Node? Next;
    }
    
    // ❌ Vulnerable to ABA
    private Node? head;
    
    public void Push(int value)
    {
        var newNode = new Node { Value = value };
        
        while (true)
        {
            var currentHead = Volatile.Read(ref head);
            newNode.Next = currentHead;
            
            // ABA problem: currentHead could be recycled!
            if (Interlocked.CompareExchange(ref head, newNode, currentHead) == currentHead)
                break;
        }
    }
}
```

## ABA Problem Solutions

```csharp
// ✅ Solution 1: Version Counter
public class VersionedStack<T>
{
    private readonly record struct VersionedNode(Node? Node, long Version);
    
    private class Node
    {
        public T Value;
        public Node? Next;
        
        public Node(T value) => Value = value;
    }
    
    private VersionedNode head;
    
    public void Push(T value)
    {
        var newNode = new Node(value);
        
        while (true)
        {
            var currentHead = Volatile.Read(ref head);
            newNode.Next = currentHead.Node;
            
            var newHead = new VersionedNode(newNode, currentHead.Version + 1);
            
            // Version prevents ABA: even if same node, version differs
            if (Interlocked.CompareExchange(ref head, newHead, currentHead).Equals(currentHead))
                break;
        }
    }
    
    public bool TryPop(out T value)
    {
        while (true)
        {
            var currentHead = Volatile.Read(ref head);
            
            if (currentHead.Node == null)
            {
                value = default!;
                return false;
            }
            
            var newHead = new VersionedNode(currentHead.Node.Next, currentHead.Version + 1);
            
            if (Interlocked.CompareExchange(ref head, newHead, currentHead).Equals(currentHead))
            {
                value = currentHead.Node.Value;
                return true;
            }
        }
    }
}

// ✅ Solution 2: Hazard Pointers (prevent node reuse)
public class HazardPointerStack<T>
{
    private class Node
    {
        public T Value;
        public Node? Next;
    }
    
    private Node? head;
    private readonly ThreadLocal<Node?> hazardPointer = new();
    
    public void Push(T value)
    {
        var newNode = new Node { Value = value };
        
        while (true)
        {
            var currentHead = Volatile.Read(ref head);
            newNode.Next = currentHead;
            
            if (Interlocked.CompareExchange(ref head, newNode, currentHead) == currentHead)
                break;
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
            
            // Set hazard pointer: tells other threads not to recycle this node
            hazardPointer.Value = currentHead;
            
            // Re-check head didn't change
            if (Volatile.Read(ref head) != currentHead)
                continue;
            
            if (Interlocked.CompareExchange(ref head, currentHead.Next, currentHead) == currentHead)
            {
                value = currentHead.Value;
                hazardPointer.Value = null;  // Clear hazard
                return true;
            }
        }
    }
}

// ✅ Solution 3: Use managed memory (GC prevents reuse)
public class ManagedStack<T>
{
    private class Node
    {
        public T Value;
        public Node? Next;
        
        public Node(T value) => Value = value;
    }
    
    private Node? head;
    
    public void Push(T value)
    {
        var newNode = new Node(value);
        
        while (true)
        {
            var currentHead = Volatile.Read(ref head);
            newNode.Next = currentHead;
            
            // GC keeps nodes alive, so no reuse → no ABA
            if (Interlocked.CompareExchange(ref head, newNode, currentHead) == currentHead)
                break;
        }
    }
}
```

## Double-Width CAS

```csharp
// ✅ 128-bit CAS on 64-bit systems (if supported)
[StructLayout(LayoutKind.Sequential, Pack = 16)]
public struct DoubleWidth
{
    public long Value1;
    public long Value2;
}

public class DoubleWidthCAS
{
    // Some platforms support 128-bit CAS (CMPXCHG16B on x64)
    // .NET doesn't directly expose this, but could use via P/Invoke
    
    // ✅ Alternative: pack into single value
    public readonly record struct PackedValue(long High, long Low)
    {
        public static explicit operator long(PackedValue p)
        {
            return (p.High << 32) | (p.Low & 0xFFFFFFFF);
        }
        
        public static explicit operator PackedValue(long value)
        {
            return new PackedValue(value >> 32, value & 0xFFFFFFFF);
        }
    }
}
```

## CAS-Based Data Structures

```csharp
// ✅ CAS-based linked list insert
public class CASLinkedList<T>
{
    private class Node
    {
        public T Value;
        public Node? Next;
        
        public Node(T value) => Value = value;
    }
    
    private Node? head;
    
    public void Insert(T value)
    {
        var newNode = new Node(value);
        
        while (true)
        {
            var currentHead = Volatile.Read(ref head);
            newNode.Next = currentHead;
            
            if (Interlocked.CompareExchange(ref head, newNode, currentHead) == currentHead)
                return;
        }
    }
    
    public bool Remove(T value)
    {
        while (true)
        {
            var prev = (Node?)null;
            var current = Volatile.Read(ref head);
            
            while (current != null)
            {
                if (EqualityComparer<T>.Default.Equals(current.Value, value))
                {
                    var next = current.Next;
                    
                    if (prev == null)
                    {
                        // Remove head
                        if (Interlocked.CompareExchange(ref head, next, current) == current)
                            return true;
                    }
                    else
                    {
                        // Remove middle/tail
                        if (Interlocked.CompareExchange(ref prev.Next, next, current) == current)
                            return true;
                    }
                    
                    break;  // Retry from start
                }
                
                prev = current;
                current = current.Next;
            }
            
            if (current == null)
                return false;  // Not found
        }
    }
}
```

## CAS Performance Optimization

```csharp
public class OptimizedCAS
{
    private long value;
    
    // ✅ Fast path: optimistic read
    public long IncrementOptimistic()
    {
        // Try once without retry loop (fast path)
        long current = Volatile.Read(ref value);
        long next = current + 1;
        
        if (Interlocked.CompareExchange(ref value, next, current) == current)
            return next;  // Success on first try!
        
        // Slow path: retry loop
        while (true)
        {
            current = Volatile.Read(ref value);
            next = current + 1;
            
            if (Interlocked.CompareExchange(ref value, next, current) == current)
                return next;
        }
    }
    
    // ✅ Batch operations reduce CAS contention
    public void AddBatch(long[] values)
    {
        long sum = 0;
        foreach (long v in values)
        {
            sum += v;
        }
        
        // Single CAS for entire batch
        long current;
        do
        {
            current = Volatile.Read(ref value);
        }
        while (Interlocked.CompareExchange(ref value, current + sum, current) != current);
    }
}
```

## Best Practices

### 1. Keep CAS Loop Simple

```csharp
// ✅ Simple, fast retry loop
long current, next;
do
{
    current = value;
    next = current + 1;
}
while (Interlocked.CompareExchange(ref value, next, current) != current);

// ❌ Don't do expensive work in loop
do
{
    current = value;
    next = ExpensiveComputation(current);  // Computed multiple times!
}
while (Interlocked.CompareExchange(ref value, next, current) != current);
```

### 2. Use Backoff Under Contention

```csharp
// ✅ Exponential backoff
int retries = 0;
while (CAS fails)
{
    if (retries++ > 10)
        Thread.SpinWait(1 << Math.Min(retries - 10, 10));
}
```

### 3. Avoid ABA Problem

```csharp
// ✅ Use version counter
private record struct Versioned<T>(T Value, long Version);

// ✅ Or use managed memory (GC prevents reuse)
```

### 4. Provide Non-CAS Alternative

```csharp
// ✅ Fallback when CAS unavailable
public void Update(int value)
{
    if (CanUseCAS())
        UpdateCAS(value);
    else
        UpdateLocked(value);
}
```

## Performance Characteristics

- **CAS operation**: 20-100 cycles
- **Retry overhead**: Depends on contention
- **Low contention**: Near zero retries
- **High contention**: Many retries (use backoff)

## When to Use

- Lock-free data structures
- Atomic multi-field updates
- Optimistic concurrency
- High-performance counters
- State machines

## When NOT to Use

- Complex multi-step updates (use locks)
- High contention (consider locks)
- Need for fairness (CAS doesn't guarantee)

## Related Patterns

- [Atomic Operations Patterns](./atomic-operations-patterns.md) — Interlocked operations
- [Lock-Free Data Structures](./lock-free-data-structures.md) — Lock-free algorithms
- [Memory Barriers](./memory-barriers-synchronization.md) — Memory ordering
- [Lock-Free vs Wait-Free](./lock-free-vs-wait-free.md) — Progress guarantees

## References

- "The Art of Multiprocessor Programming" by Herlihy & Shavit
- "Interlocked.CompareExchange" - Microsoft Docs
- "ABA Problem" - Wikipedia
- "Lock-Free Programming" - Preshing on Programming
