# Testing Timeouts and Delays

> Verify asynchronous operations complete within expected time limits and test timeout behavior using FluentAssertions.

## Problem

Asynchronous code may have time constraints, and tests need to verify operations complete within acceptable timeframes, handle timeouts gracefully, and test delay behavior without slowing down test suites.

## Pattern

Use FluentAssertions' timing assertions and async testing capabilities to verify execution time and timeout behavior.

## Example

### ❌ Before - Manual Timing Checks

```csharp
[TestMethod]
public async Task ProcessAsync_CompletesQuickly()
{
    var start = DateTime.Now;
    await processor.ProcessAsync(order);
    var elapsed = DateTime.Now - start;
    Assert.IsTrue(elapsed.TotalSeconds < 5);
    // ❌ Imprecise and doesn't test timeout behavior
}
```

### ✅ After - FluentAssertions Timing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class ProcessorTimingTests
{
    [TestMethod]
    public async Task ProcessAsync_CompletesWithinTimeout()
    {
        // Arrange
        var processor = new OrderProcessor();
        var order = CreateOrder();

        // Act
        Func<Task> act = async () => await processor.ProcessAsync(order);

        // Assert
        await act.Should().CompleteWithinAsync(TimeSpan.FromSeconds(5));
    }

    [TestMethod]
    public async Task ProcessAsync_WithTimeout_ThrowsWhenExceeded()
    {
        // Arrange
        var processor = new SlowOrderProcessor();
        var order = CreateOrder();
        var timeout = TimeSpan.FromMilliseconds(100);

        // Act
        Func<Task> act = async () => 
            await processor.ProcessAsync(order, timeout);

        // Assert
        await act.Should().ThrowAsync<TimeoutException>();
    }
}
```

## Testing Completion Time

```csharp
public class CacheService
{
    public async Task<T> GetOrCreateAsync<T>(
        string key,
        Func<Task<T>> factory,
        TimeSpan timeout)
    {
        using var cts = new CancellationTokenSource(timeout);
        return await factory().WaitAsync(cts.Token);
    }
}

[TestClass]
public class CompletionTimeTests
{
    [TestMethod]
    public async Task GetOrCreateAsync_FastOperation_CompletesQuickly()
    {
        // Arrange
        var cache = new CacheService();
        var factory = () => Task.FromResult(42);

        // Act
        Func<Task> act = async () => 
            await cache.GetOrCreateAsync("key", factory, TimeSpan.FromSeconds(5));

        // Assert
        await act.Should().CompleteWithinAsync(TimeSpan.FromMilliseconds(100));
    }

    [TestMethod]
    public async Task GetOrCreateAsync_CompletesFasterThanTimeout()
    {
        // Arrange
        var cache = new CacheService();
        var factory = async () =>
        {
            await Task.Delay(50);
            return 42;
        };

        // Act
        Func<Task> act = async () =>
            await cache.GetOrCreateAsync("key", factory, TimeSpan.FromSeconds(1));

        // Assert
        await act.Should().CompleteWithinAsync(TimeSpan.FromSeconds(1));
    }
}
```

## Testing Timeout Exceptions

```csharp
public class ApiClient
{
    private readonly HttpClient _httpClient;
    private readonly TimeSpan _defaultTimeout = TimeSpan.FromSeconds(30);

    public async Task<Response> GetAsync(string url, TimeSpan? timeout = null)
    {
        using var cts = new CancellationTokenSource(timeout ?? _defaultTimeout);
        try
        {
            var response = await _httpClient.GetAsync(url, cts.Token);
            return await ParseResponseAsync(response);
        }
        catch (OperationCanceledException) when (cts.Token.IsCancellationRequested)
        {
            throw new TimeoutException($"Request to {url} timed out");
        }
    }
}

[TestClass]
public class TimeoutExceptionTests
{
    [TestMethod]
    public async Task GetAsync_WhenTimeout_ThrowsTimeoutException()
    {
        // Arrange
        var client = CreateClientWithSlowServer();
        var url = "https://slow-api.example.com/data";

        // Act
        Func<Task> act = async () => 
            await client.GetAsync(url, TimeSpan.FromMilliseconds(10));

        // Assert
        await act.Should()
            .ThrowAsync<TimeoutException>()
            .WithMessage("*timed out*");
    }

    [TestMethod]
    public async Task GetAsync_WithinTimeout_Succeeds()
    {
        // Arrange
        var client = CreateClientWithFastServer();
        var url = "https://fast-api.example.com/data";

        // Act
        Func<Task> act = async () =>
            await client.GetAsync(url, TimeSpan.FromSeconds(5));

        // Assert
        await act.Should().NotThrowAsync();
    }
}
```

## Testing Delay Behavior

```csharp
public class RetryPolicy
{
    public async Task<T> ExecuteAsync<T>(
        Func<Task<T>> operation,
        int maxRetries,
        TimeSpan delay)
    {
        for (int i = 0; i < maxRetries; i++)
        {
            try
            {
                return await operation();
            }
            catch when (i < maxRetries - 1)
            {
                await Task.Delay(delay);
            }
        }
        throw new InvalidOperationException("Max retries exceeded");
    }
}

[TestClass]
public class DelayBehaviorTests
{
    [TestMethod]
    public async Task ExecuteAsync_WithRetries_TakesExpectedTime()
    {
        // Arrange
        var policy = new RetryPolicy();
        var attempts = 0;
        var delay = TimeSpan.FromMilliseconds(100);

        Func<Task<int>> operation = () =>
        {
            attempts++;
            if (attempts < 3)
                throw new Exception("Temporary failure");
            return Task.FromResult(42);
        };

        // Act
        Func<Task> act = async () => 
            await policy.ExecuteAsync(operation, maxRetries: 3, delay);

        // Assert
        // 2 retries with 100ms delay each = ~200ms minimum
        await act.Should().CompleteWithinAsync(TimeSpan.FromSeconds(1));
        attempts.Should().Be(3);
    }

    [TestMethod]
    public async Task ExecuteAsync_ImmediateSuccess_NoDelay()
    {
        // Arrange
        var policy = new RetryPolicy();
        Func<Task<int>> operation = () => Task.FromResult(42);

        // Act
        Func<Task> act = async () =>
            await policy.ExecuteAsync(operation, maxRetries: 3, TimeSpan.FromSeconds(10));

        // Assert
        // Should complete immediately without delay
        await act.Should().CompleteWithinAsync(TimeSpan.FromMilliseconds(100));
    }
}
```

## Testing Cancellation Token Timeout

```csharp
public class DataProcessor
{
    public async Task<ProcessedData> ProcessAsync(
        Data data,
        CancellationToken cancellationToken)
    {
        for (int i = 0; i < data.Items.Count; i++)
        {
            cancellationToken.ThrowIfCancellationRequested();
            await ProcessItemAsync(data.Items[i], cancellationToken);
        }
        return new ProcessedData(data);
    }
}

[TestClass]
public class CancellationTokenTimeoutTests
{
    [TestMethod]
    public async Task ProcessAsync_WithCancellation_ThrowsOperationCanceled()
    {
        // Arrange
        var processor = new DataProcessor();
        var data = CreateLargeDataSet();

        using var cts = new CancellationTokenSource(TimeSpan.FromMilliseconds(10));

        // Act
        Func<Task> act = async () =>
            await processor.ProcessAsync(data, cts.Token);

        // Assert
        await act.Should()
            .ThrowAsync<OperationCanceledException>()
            .Where(ex => ex.CancellationToken == cts.Token);
    }

    [TestMethod]
    public async Task ProcessAsync_CompletesBeforeCancellation()
    {
        // Arrange
        var processor = new DataProcessor();
        var data = CreateSmallDataSet();

        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

        // Act
        Func<Task> act = async () =>
            await processor.ProcessAsync(data, cts.Token);

        // Assert
        await act.Should().NotThrowAsync();
    }
}
```

## Testing Task.WaitAsync (C# 10+)

```csharp
[TestClass]
public class WaitAsyncTests
{
    [TestMethod]
    public async Task WaitAsync_CompletesWithinTimeout()
    {
        // Arrange
        var task = Task.Delay(50).ContinueWith(_ => 42);

        // Act
        var result = await task.WaitAsync(TimeSpan.FromSeconds(1));

        // Assert
        result.Should().Be(42);
    }

    [TestMethod]
    public async Task WaitAsync_ExceedsTimeout_ThrowsTimeoutException()
    {
        // Arrange
        var task = Task.Delay(1000).ContinueWith(_ => 42);

        // Act
        Func<Task> act = async () =>
            await task.WaitAsync(TimeSpan.FromMilliseconds(10));

        // Assert
        await act.Should().ThrowAsync<TimeoutException>();
    }
}
```

## Testing Stopwatch and Precise Timing

```csharp
public class PerformanceMonitor
{
    public async Task<(T Result, TimeSpan Elapsed)> MeasureAsync<T>(Func<Task<T>> operation)
    {
        var stopwatch = Stopwatch.StartNew();
        var result = await operation();
        stopwatch.Stop();
        return (result, stopwatch.Elapsed);
    }
}

[TestClass]
public class PreciseTimingTests
{
    [TestMethod]
    public async Task MeasureAsync_ReturnsAccurateElapsedTime()
    {
        // Arrange
        var monitor = new PerformanceMonitor();
        var expectedDelay = TimeSpan.FromMilliseconds(100);

        // Act
        var (result, elapsed) = await monitor.MeasureAsync(async () =>
        {
            await Task.Delay(expectedDelay);
            return 42;
        });

        // Assert
        result.Should().Be(42);
        elapsed.Should().BeCloseTo(expectedDelay, precision: TimeSpan.FromMilliseconds(50));
    }

    [TestMethod]
    public async Task MeasureAsync_FastOperation_MeasuresAccurately()
    {
        // Arrange
        var monitor = new PerformanceMonitor();

        // Act
        var (result, elapsed) = await monitor.MeasureAsync(() => Task.FromResult(42));

        // Assert
        result.Should().Be(42);
        elapsed.Should().BeLessThan(TimeSpan.FromMilliseconds(10));
    }
}
```

## Testing Deadline-Based Operations

```csharp
public class DeadlineService
{
    public async Task<Result> ExecuteWithDeadlineAsync(
        Func<Task> operation,
        DateTime deadline)
    {
        var timeUntilDeadline = deadline - DateTime.Now;
        if (timeUntilDeadline <= TimeSpan.Zero)
            return Result.Failure("Deadline already passed");

        using var cts = new CancellationTokenSource(timeUntilDeadline);

        try
        {
            await operation();
            return Result.Success();
        }
        catch (OperationCanceledException)
        {
            return Result.Failure("Deadline exceeded");
        }
    }
}

[TestClass]
public class DeadlineTests
{
    [TestMethod]
    public async Task ExecuteWithDeadlineAsync_BeforeDeadline_Succeeds()
    {
        // Arrange
        var service = new DeadlineService();
        var deadline = DateTime.Now.AddSeconds(5);

        // Act
        var result = await service.ExecuteWithDeadlineAsync(
            async () => await Task.Delay(10),
            deadline
        );

        // Assert
        result.IsSuccess.Should().BeTrue();
    }

    [TestMethod]
    public async Task ExecuteWithDeadlineAsync_PastDeadline_Fails()
    {
        // Arrange
        var service = new DeadlineService();
        var deadline = DateTime.Now.AddMilliseconds(-100);

        // Act
        var result = await service.ExecuteWithDeadlineAsync(
            async () => await Task.Delay(10),
            deadline
        );

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("already passed");
    }
}
```

## Why It's Important

1. **Performance SLAs**: Verify operations meet time requirements
2. **Timeout Handling**: Ensure graceful timeout behavior
3. **Responsiveness**: Test that UI/API remains responsive
4. **Resource Management**: Prevent operations from running indefinitely
5. **Reliability**: Ensure consistent timing behavior

## Guidelines

**Timeout Testing**
- `.Should().CompleteWithinAsync(timeSpan)` - verify completion time
- `.Should().ThrowAsync<TimeoutException>()` - verify timeout throws
- Use `CancellationTokenSource` with timeout for cancellation
- Test both success and timeout scenarios

**Timing Assertions**
- Use `.BeCloseTo()` for approximate timing (accounts for variance)
- Provide reasonable precision tolerances
- Test in milliseconds for precise operations
- Use seconds for integration tests

**Best Practices**
- Don't make tests dependent on exact timing
- Use generous timeouts in assertions to avoid flaky tests
- Test timeout behavior separately from happy path
- Use `Task.Delay` sparingly in tests (prefer mocks)

## Common Pitfalls

❌ **Too strict timing requirements**
```csharp
// Flaky - fails if system is slow
elapsed.Should().Be(TimeSpan.FromMilliseconds(100));
```

✅ **Use tolerance**
```csharp
// Robust
elapsed.Should().BeCloseTo(
    TimeSpan.FromMilliseconds(100),
    precision: TimeSpan.FromMilliseconds(50)
);
```

❌ **Testing delays with real delays**
```csharp
// Slow test
await Task.Delay(5000); // Makes tests slow!
```

✅ **Mock time-dependent operations**
```csharp
// Fast test
var mockTimeProvider = CreateMockWithDelay();
await service.ExecuteAsync(mockTimeProvider);
```

## See Also

- [Testing Async Code](./testing-async-code.md) — async testing patterns
- [Testing Concurrency](./testing-concurrency.md) — concurrent operations
- [Cancellation Token Handling](../patterns/cancellation-token-handling.md) — cancellation patterns
- [Async Patterns](../patterns/async-patterns.md) — async/await patterns
