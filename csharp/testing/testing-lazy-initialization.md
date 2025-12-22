# Testing Lazy Initialization

> Test lazy-loaded values and Lazy<T>—verify deferred initialization, thread safety, and memoization.

## Problem

Lazy initialization delays expensive object creation until needed. Tests must verify initialization happens once, at the right time, and is thread-safe.

## Pattern

Test that lazy values are not initialized until accessed, initialization happens exactly once, and concurrent access is safe.

## Example

### ❌ Before - Eager Initialization

```csharp
public class ExpensiveService
{
    private readonly ExpensiveResource resource;  // Always created
    
    public ExpensiveService()
    {
        resource = new ExpensiveResource();  // Created even if never used
    }
}
```

### ✅ After - Lazy Initialization

```csharp
public class ExpensiveService
{
    private readonly Lazy<ExpensiveResource> resource;
    
    public ExpensiveService()
    {
        resource = new Lazy<ExpensiveResource>(() => new ExpensiveResource());
    }
    
    public ExpensiveResource Resource => resource.Value;
}

[Fact]
public void Resource_NotAccessed_NotInitialized()
{
    // Arrange
    var service = new ExpensiveService();
    
    // Assert
    service.IsResourceInitialized.Should().BeFalse();
}

[Fact]
public void Resource_FirstAccess_InitializesOnce()
{
    // Arrange
    var service = new ExpensiveService();
    
    // Act
    var resource1 = service.Resource;
    var resource2 = service.Resource;
    
    // Assert
    resource1.Should().BeSameAs(resource2);
    service.InitializationCount.Should().Be(1);
}
```

## Testing Lazy<T>

### Basic Lazy Initialization

```csharp
[Fact]
public void LazyValue_BeforeAccess_IsNotCreated()
{
    // Arrange
    var initializationCount = 0;
    var lazy = new Lazy<string>(() =>
    {
        initializationCount++;
        return "value";
    });
    
    // Assert
    lazy.IsValueCreated.Should().BeFalse();
    initializationCount.Should().Be(0);
}

[Fact]
public void LazyValue_AfterAccess_IsCreatedOnce()
{
    // Arrange
    var initializationCount = 0;
    var lazy = new Lazy<string>(() =>
    {
        initializationCount++;
        return "value";
    });
    
    // Act
    var value1 = lazy.Value;
    var value2 = lazy.Value;
    
    // Assert
    lazy.IsValueCreated.Should().BeTrue();
    initializationCount.Should().Be(1);
    value1.Should().Be("value");
    value2.Should().BeSameAs(value1);
}
```

## Testing Thread Safety

### Concurrent Access

```csharp
[Fact]
public void LazyValue_ConcurrentAccess_InitializesOnce()
{
    // Arrange
    var initializationCount = 0;
    var lazy = new Lazy<ExpensiveResource>(() =>
    {
        Interlocked.Increment(ref initializationCount);
        Thread.Sleep(100);  // Simulate slow initialization
        return new ExpensiveResource();
    }, LazyThreadSafetyMode.ExecutionAndPublication);
    
    var results = new ConcurrentBag<ExpensiveResource>();
    
    // Act
    Parallel.For(0, 10, _ =>
    {
        results.Add(lazy.Value);
    });
    
    // Assert
    initializationCount.Should().Be(1);
    results.Distinct().Should().HaveCount(1);
}
```

## Testing Lazy Initialization Modes

### Publication Only

```csharp
[Fact]
public void LazyValue_PublicationOnly_MayInitializeMultipleTimes()
{
    // Arrange
    var initializationCount = 0;
    var lazy = new Lazy<string>(() =>
    {
        Interlocked.Increment(ref initializationCount);
        return "value";
    }, LazyThreadSafetyMode.PublicationOnly);
    
    // Act
    Parallel.For(0, 10, _ =>
    {
        var _ = lazy.Value;
    });
    
    // Assert
    initializationCount.Should().BeGreaterOrEqualTo(1);
}
```

## Testing Custom Lazy Initialization

### Manual Lazy Pattern

```csharp
public class CachedCalculator
{
    private readonly object lockObject = new();
    private Result? cachedResult;
    
    public Result GetResult()
    {
        if (cachedResult == null)
        {
            lock (lockObject)
            {
                if (cachedResult == null)
                {
                    cachedResult = PerformExpensiveCalculation();
                }
            }
        }
        return cachedResult;
    }
}

[Fact]
public void GetResult_CalledMultipleTimes_CalculatesOnce()
{
    // Arrange
    var calculator = new CachedCalculator();
    
    // Act
    var result1 = calculator.GetResult();
    var result2 = calculator.GetResult();
    
    // Assert
    result1.Should().BeSameAs(result2);
    calculator.CalculationCount.Should().Be(1);
}
```

## Testing Exception Handling

### Lazy Initialization Failures

```csharp
[Fact]
public void LazyValue_InitializationThrows_ThrowsOnEveryAccess()
{
    // Arrange
    var lazy = new Lazy<string>(() => throw new InvalidOperationException("Init failed"));
    
    // Act & Assert
    Action act1 = () => { var _ = lazy.Value; };
    Action act2 = () => { var _ = lazy.Value; };
    
    act1.Should().Throw<InvalidOperationException>().WithMessage("Init failed");
    act2.Should().Throw<InvalidOperationException>().WithMessage("Init failed");
    lazy.IsValueCreated.Should().BeFalse();
}
```

## See Also

- [Testing Concurrency](./testing-concurrency.md) — thread-safe initialization
- [Testing Disposable Resources](./testing-disposable-resources.md) — lazy disposable objects
- [Testing Static Methods](./testing-static-methods.md) — singleton patterns
