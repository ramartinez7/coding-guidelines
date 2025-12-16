# Testing Concurrency

> Verify thread-safe operations, concurrent access patterns, and race conditions using MSTest and FluentAssertions.

## Problem

Concurrent code is difficult to test because race conditions may only manifest under specific timing conditions. Tests must verify thread safety, detect race conditions, and ensure proper synchronization.

## Pattern

Test concurrent operations using multiple tasks, verify thread safety with concurrent access, and use synchronization primitives to detect race conditions.

## Example

### ❌ Before - No Concurrency Testing

```csharp
[TestMethod]
public void Increment_IncreasesCounter()
{
    var counter = new Counter();
    counter.Increment();
    Assert.AreEqual(1, counter.Value);
    // ❌ Doesn't test thread safety
}
```

### ✅ After - Concurrent Access Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

public class ThreadSafeCounter
{
    private int _value;

    public int Value => _value;

    public void Increment() => Interlocked.Increment(ref _value);
}

[TestClass]
public class ConcurrencyTests
{
    [TestMethod]
    public async Task Increment_ConcurrentAccess_IsThreadSafe()
    {
        // Arrange
        var counter = new ThreadSafeCounter();
        var iterations = 1000;
        var taskCount = 10;

        // Act
        var tasks = Enumerable.Range(0, taskCount)
            .Select(_ => Task.Run(() =>
            {
                for (int i = 0; i < iterations; i++)
                {
                    counter.Increment();
                }
            }));

        await Task.WhenAll(tasks);

        // Assert
        counter.Value.Should().Be(taskCount * iterations);
    }

    [TestMethod]
    public async Task Increment_HighConcurrency_MaintainsConsistency()
    {
        // Arrange
        var counter = new ThreadSafeCounter();
        var concurrentTasks = 100;

        // Act
        var tasks = Enumerable.Range(0, concurrentTasks)
            .Select(_ => Task.Run(() => counter.Increment()));

        await Task.WhenAll(tasks);

        // Assert
        counter.Value.Should().Be(concurrentTasks);
    }
}
```

## Testing Thread-Safe Collections

```csharp
public class ThreadSafeCache<TKey, TValue> where TKey : notnull
{
    private readonly ConcurrentDictionary<TKey, TValue> _cache = new();

    public bool TryAdd(TKey key, TValue value) => 
        _cache.TryAdd(key, value);

    public bool TryGet(TKey key, out TValue? value) =>
        _cache.TryGetValue(key, out value);

    public int Count => _cache.Count;
}

[TestClass]
public class ThreadSafeCacheTests
{
    [TestMethod]
    public async Task TryAdd_ConcurrentAccess_AllItemsAdded()
    {
        // Arrange
        var cache = new ThreadSafeCache<int, string>();
        var itemsPerTask = 100;
        var taskCount = 10;

        // Act
        var tasks = Enumerable.Range(0, taskCount)
            .Select(taskId => Task.Run(() =>
            {
                for (int i = 0; i < itemsPerTask; i++)
                {
                    var key = taskId * itemsPerTask + i;
                    cache.TryAdd(key, $"Value_{key}");
                }
            }));

        await Task.WhenAll(tasks);

        // Assert
        cache.Count.Should().Be(taskCount * itemsPerTask);
    }

    [TestMethod]
    public async Task TryGet_ConcurrentReads_ReturnsCorrectValues()
    {
        // Arrange
        var cache = new ThreadSafeCache<int, string>();
        for (int i = 0; i < 100; i++)
        {
            cache.TryAdd(i, $"Value_{i}");
        }

        // Act
        var tasks = Enumerable.Range(0, 100)
            .Select(key => Task.Run(() =>
            {
                cache.TryGet(key, out var value);
                return value;
            }));

        var results = await Task.WhenAll(tasks);

        // Assert
        results.Should().AllSatisfy(result => result.Should().NotBeNull());
    }
}
```

## Testing Lock-Based Synchronization

```csharp
public class BankAccount
{
    private readonly object _lock = new();
    private decimal _balance;

    public decimal Balance
    {
        get
        {
            lock (_lock)
            {
                return _balance;
            }
        }
    }

    public void Deposit(decimal amount)
    {
        lock (_lock)
        {
            _balance += amount;
        }
    }

    public bool Withdraw(decimal amount)
    {
        lock (_lock)
        {
            if (_balance >= amount)
            {
                _balance -= amount;
                return true;
            }
            return false;
        }
    }
}

[TestClass]
public class LockSynchronizationTests
{
    [TestMethod]
    public async Task Deposit_ConcurrentDeposits_CorrectBalance()
    {
        // Arrange
        var account = new BankAccount();
        var depositAmount = 10m;
        var depositCount = 1000;

        // Act
        var tasks = Enumerable.Range(0, depositCount)
            .Select(_ => Task.Run(() => account.Deposit(depositAmount)));

        await Task.WhenAll(tasks);

        // Assert
        account.Balance.Should().Be(depositAmount * depositCount);
    }

    [TestMethod]
    public async Task Withdraw_ConcurrentWithdraws_NeverNegative()
    {
        // Arrange
        var account = new BankAccount();
        account.Deposit(1000m);

        // Act
        var tasks = Enumerable.Range(0, 100)
            .Select(_ => Task.Run(() => account.Withdraw(20m)));

        await Task.WhenAll(tasks);

        // Assert
        account.Balance.Should().BeGreaterThanOrEqualTo(0);
    }
}
```

## Testing SemaphoreSlim

```csharp
public class RateLimiter
{
    private readonly SemaphoreSlim _semaphore;
    private readonly int _maxConcurrent;

    public RateLimiter(int maxConcurrent)
    {
        _maxConcurrent = maxConcurrent;
        _semaphore = new SemaphoreSlim(maxConcurrent, maxConcurrent);
    }

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation)
    {
        await _semaphore.WaitAsync();
        try
        {
            return await operation();
        }
        finally
        {
            _semaphore.Release();
        }
    }
}

[TestClass]
public class RateLimiterTests
{
    [TestMethod]
    public async Task ExecuteAsync_EnforcesConcurrencyLimit()
    {
        // Arrange
        var limiter = new RateLimiter(maxConcurrent: 3);
        var concurrentCount = 0;
        var maxObservedConcurrent = 0;
        var lockObject = new object();

        // Act
        var tasks = Enumerable.Range(0, 10)
            .Select(_ => limiter.ExecuteAsync(async () =>
            {
                lock (lockObject)
                {
                    concurrentCount++;
                    maxObservedConcurrent = Math.Max(maxObservedConcurrent, concurrentCount);
                }

                await Task.Delay(50);

                lock (lockObject)
                {
                    concurrentCount--;
                }

                return true;
            }));

        await Task.WhenAll(tasks);

        // Assert
        maxObservedConcurrent.Should().BeLessThanOrEqualTo(3);
    }
}
```

## Testing ReaderWriterLockSlim

```csharp
public class CachedRepository<T>
{
    private readonly ReaderWriterLockSlim _lock = new();
    private readonly Dictionary<string, T> _cache = new();

    public T? Read(string key)
    {
        _lock.EnterReadLock();
        try
        {
            return _cache.TryGetValue(key, out var value) ? value : default;
        }
        finally
        {
            _lock.ExitReadLock();
        }
    }

    public void Write(string key, T value)
    {
        _lock.EnterWriteLock();
        try
        {
            _cache[key] = value;
        }
        finally
        {
            _lock.ExitWriteLock();
        }
    }
}

[TestClass]
public class ReaderWriterLockTests
{
    [TestMethod]
    public async Task ReadWrite_ConcurrentAccess_Consistent()
    {
        // Arrange
        var repository = new CachedRepository<string>();
        repository.Write("key1", "initial");

        // Act - Concurrent reads and writes
        var readTasks = Enumerable.Range(0, 100)
            .Select(_ => Task.Run(() => repository.Read("key1")));

        var writeTasks = Enumerable.Range(0, 10)
            .Select(i => Task.Run(() => repository.Write("key1", $"value_{i}")));

        await Task.WhenAll(readTasks.Concat(writeTasks));

        // Assert - No assertion failure means thread safety maintained
        var finalValue = repository.Read("key1");
        finalValue.Should().NotBeNull();
    }
}
```

## Testing Async/Await Concurrency

```csharp
public class AsyncProcessor
{
    private int _processedCount;

    public async Task<int> ProcessBatchAsync(IEnumerable<int> items)
    {
        var tasks = items.Select(ProcessItemAsync);
        await Task.WhenAll(tasks);
        return _processedCount;
    }

    private async Task ProcessItemAsync(int item)
    {
        await Task.Delay(1); // Simulate async work
        Interlocked.Increment(ref _processedCount);
    }
}

[TestClass]
public class AsyncConcurrencyTests
{
    [TestMethod]
    public async Task ProcessBatchAsync_ConcurrentProcessing_CountsAllItems()
    {
        // Arrange
        var processor = new AsyncProcessor();
        var items = Enumerable.Range(1, 100);

        // Act
        var count = await processor.ProcessBatchAsync(items);

        // Assert
        count.Should().Be(100);
    }

    [TestMethod]
    public async Task ProcessBatchAsync_MultipleBatches_Concurrent()
    {
        // Arrange
        var processor = new AsyncProcessor();

        // Act
        var batch1 = processor.ProcessBatchAsync(Enumerable.Range(1, 50));
        var batch2 = processor.ProcessBatchAsync(Enumerable.Range(51, 50));

        await Task.WhenAll(batch1, batch2);

        // Assert - Both batches processed
        (await batch1 + await batch2).Should().Be(100);
    }
}
```

## Testing Race Conditions

```csharp
public class LazyInitializer<T> where T : class, new()
{
    private T? _instance;
    private readonly object _lock = new();

    public T Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                    {
                        _instance = new T();
                    }
                }
            }
            return _instance;
        }
    }
}

[TestClass]
public class RaceConditionTests
{
    [TestMethod]
    public async Task Instance_ConcurrentAccess_CreatesSingleInstance()
    {
        // Arrange
        var initializer = new LazyInitializer<object>();

        // Act - Concurrent access to trigger potential race
        var tasks = Enumerable.Range(0, 1000)
            .Select(_ => Task.Run(() => initializer.Instance));

        var instances = await Task.WhenAll(tasks);

        // Assert - All references should be the same instance
        var firstInstance = instances.First();
        instances.Should().AllSatisfy(instance =>
            ReferenceEquals(instance, firstInstance).Should().BeTrue()
        );
    }
}
```

## Testing Parallel.ForEach

```csharp
public class ParallelProcessor
{
    public int ProcessParallel(IEnumerable<int> items)
    {
        var count = 0;

        Parallel.ForEach(items, item =>
        {
            Interlocked.Increment(ref count);
        });

        return count;
    }
}

[TestClass]
public class ParallelProcessingTests
{
    [TestMethod]
    public void ProcessParallel_AllItemsProcessed()
    {
        // Arrange
        var processor = new ParallelProcessor();
        var items = Enumerable.Range(1, 1000);

        // Act
        var count = processor.ProcessParallel(items);

        // Assert
        count.Should().Be(1000);
    }
}
```

## Why It's Important

1. **Thread Safety**: Verify code works correctly under concurrent access
2. **Race Conditions**: Detect and prevent race conditions
3. **Deadlocks**: Ensure proper lock ordering
4. **Performance**: Test scalability with concurrent operations
5. **Correctness**: Guarantee data consistency

## Guidelines

**Testing Concurrent Code**
- Use multiple tasks/threads to create concurrent scenarios
- Test with high concurrency to increase chance of race conditions
- Verify final state consistency
- Use atomic operations (Interlocked) for counters
- Test both success and contention scenarios

**Synchronization Primitives**
- Test lock-based code with concurrent access
- Verify semaphores enforce limits
- Test reader/writer locks with concurrent reads/writes
- Ensure proper lock disposal

**Common Patterns**
- `Task.WhenAll()` for concurrent task execution
- `Parallel.ForEach()` for parallel iterations
- `Interlocked` for thread-safe operations
- `ConcurrentDictionary` for thread-safe collections

## Common Pitfalls

❌ **Not testing enough concurrency**
```csharp
// Only 2 tasks - may not expose race conditions
var task1 = Task.Run(() => counter.Increment());
var task2 = Task.Run(() => counter.Increment());
```

✅ **Test with high concurrency**
```csharp
// Many tasks increase chance of finding issues
var tasks = Enumerable.Range(0, 1000)
    .Select(_ => Task.Run(() => counter.Increment()));
await Task.WhenAll(tasks);
```

❌ **Using non-thread-safe code**
```csharp
// Race condition!
_counter++; // Not thread-safe
```

✅ **Use atomic operations**
```csharp
// Thread-safe
Interlocked.Increment(ref _counter);
```

## See Also

- [Testing Async Code](./testing-async-code.md) — async testing
- [Testing Timeouts and Delays](./testing-timeouts-delays.md) — timing tests
- [Async Patterns](../patterns/async-patterns.md) — async/await patterns
- [Thread Safety](../patterns/memory-safety-span.md) — thread-safe patterns
