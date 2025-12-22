# High-Performance and Trading Patterns

> Patterns, practices, principles, and techniques for building high-performance, low-latency systems in C# with a focus on trading applications.

This catalog covers specialized patterns for performance-critical code, particularly in financial trading systems where microsecond-level latencies matter. These patterns prioritize predictability, throughput, and minimal allocations.

## Memory Management & Performance

### [Lock-Free Data Structures](./lock-free-data-structures.md)

Implement concurrent data structures without locks using atomic operations and compare-and-swap—eliminate contention and achieve wait-free guarantees for critical paths.

### [Memory Pooling Strategies](./memory-pooling-strategies.md)

Reuse objects instead of allocating new ones—use object pools, array pools, and memory pools to eliminate GC pressure in hot paths.

### [Zero-Copy Techniques](./zero-copy-techniques.md)

Avoid copying data between buffers using Span<T>, Memory<T>, and ref returns—minimize memory bandwidth and improve cache efficiency.

### [Cache Line Optimization](./cache-line-optimization.md)

Organize data structures to minimize cache misses and false sharing—align hot fields to cache boundaries for maximum throughput.

### [Struct and Value Types for Performance](./struct-value-types-performance.md)

Use value types strategically to reduce heap allocations and improve locality—understanding when structs help vs when they hurt performance.

## Low-Latency Patterns

### [Latency-Sensitive Design](./latency-sensitive-design.md)

Design systems for consistent, predictable low latency—measure p99/p999 latencies and eliminate tail latency through careful design.

### [Hot Path Optimization](./hot-path-optimization.md)

Identify and optimize the critical execution path—remove allocations, branches, and virtual calls from hot loops.

### [Wait-Free Algorithms](./wait-free-algorithms.md)

Guarantee bounded completion time without blocking—use wait-free data structures for deterministic performance.

### [Busy-Wait vs Sleep](./busy-wait-vs-sleep.md)

Choose the right waiting strategy for your latency requirements—busy-wait for ultra-low latency, sleep for efficiency.

### [Predictable Performance](./predictable-performance.md)

Build systems with consistent performance characteristics—avoid GC pauses, JIT compilation delays, and OS scheduling variability.

## Trading-Specific Patterns

### [Order Book Modeling](./order-book-modeling.md)

Model order books with type-safe price levels and efficient updates—represent bid/ask spreads and maintain price-time priority.

### [Market Data Processing](./market-data-processing.md)

Process high-frequency market data streams efficiently—handle tick data, aggregate to bars, and maintain multiple timeframes.

### [Price-Time Priority](./price-time-priority.md)

Implement price-time priority matching with type-safe order queues—ensure fair order execution with deterministic matching rules.

### [FIX Protocol Types](./fix-protocol-types.md)

Model FIX protocol messages with strongly-typed fields—prevent field tag errors and ensure protocol compliance at compile time.

### [Tick Data Structures](./tick-data-structures.md)

Design efficient data structures for tick-by-tick market data—optimize for sequential writes and minimal memory footprint.

## Concurrency & Threading

### [Lock-Free Queues](./lock-free-queues.md)

Implement high-throughput producer-consumer queues without locks—use ring buffers and atomic operations for maximum performance.

### [Producer-Consumer Patterns](./producer-consumer-patterns.md)

Design efficient producer-consumer pipelines for market data—separate concerns and maximize throughput with proper buffering.

### [Thread Affinity](./thread-affinity.md)

Pin threads to specific CPU cores to reduce context switches—improve cache locality and eliminate NUMA effects.

### [SpinLock vs Mutex](./spinlock-vs-mutex.md)

Choose the right synchronization primitive for your critical section—spinlocks for short waits, mutexes for longer operations.

### [Concurrent Collections Performance](./concurrent-collections-performance.md)

Select the right concurrent collection for your use case—understand performance characteristics of ConcurrentQueue, ConcurrentBag, and channels.

---

## Performance Principles

These patterns follow key principles for high-performance systems:

1. **Measure Everything** - Profile before optimizing, track latencies continuously
2. **Minimize Allocations** - Every allocation is a future GC pause
3. **Predictability Over Speed** - Consistent p99 latency beats fast average with high variance
4. **Cache Locality** - Keep hot data close together in memory
5. **Lock-Free When Possible** - Eliminate contention in critical paths
6. **Type Safety Without Cost** - Use zero-cost abstractions for safety

## When to Use These Patterns

Apply these patterns when:
- You need microsecond-level latencies
- GC pauses are unacceptable
- You're processing high-frequency data streams
- Predictable performance is critical
- You're building trading, gaming, or real-time systems

## Related Patterns

See the main [C# Patterns Index](../patterns/__index__.md) for foundational patterns:
- [Allocation Budget](../patterns/allocation-budget.md)
- [Memory Safety (Span)](../patterns/memory-safety-span.md)
- [Struct Layout](../patterns/struct-layout.md)
- [Thread Safety Patterns](../patterns/thread-safety-patterns.md)
- [Performance Profiling](../patterns/performance-profiling-optimization.md)
