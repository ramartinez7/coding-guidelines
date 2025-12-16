# Testing Disposable Resources

> Verify proper disposal of IDisposable resources, using statements, and resource cleanup using FluentAssertions.

## Problem

Resource management is critical in .NET. Tests must verify that disposable resources are properly cleaned up, that using statements work correctly, and that disposal happens in the right order.

## Pattern

Test that IDisposable objects are disposed correctly, verify disposal order, and ensure resources are cleaned up even when exceptions occur.

## Example

### ❌ Before - No Disposal Testing

```csharp
[TestMethod]
public void CreateConnection_Works()
{
    var connection = factory.CreateConnection();
    Assert.IsNotNull(connection);
    // ❌ Doesn't verify disposal
}
```

### ✅ After - Comprehensive Disposal Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

public class DatabaseConnection : IDisposable
{
    private bool _disposed;

    public bool IsDisposed => _disposed;
    public bool IsOpen { get; private set; }

    public void Open() => IsOpen = true;

    public void Dispose()
    {
        if (!_disposed)
        {
            IsOpen = false;
            _disposed = true;
        }
    }
}

[TestClass]
public class DisposableResourceTests
{
    [TestMethod]
    public void Dispose_DisposesResource()
    {
        // Arrange
        var connection = new DatabaseConnection();

        // Act
        connection.Dispose();

        // Assert
        connection.IsDisposed.Should().BeTrue();
    }

    [TestMethod]
    public void UsingStatement_DisposesAutomatically()
    {
        // Arrange
        DatabaseConnection? connection = null;

        // Act
        using (connection = new DatabaseConnection())
        {
            connection.Open();
            connection.IsOpen.Should().BeTrue();
        }

        // Assert
        connection.IsDisposed.Should().BeTrue();
        connection.IsOpen.Should().BeFalse();
    }

    [TestMethod]
    public void UsingDeclaration_DisposesAtScopeEnd()
    {
        // Arrange
        DatabaseConnection? connection = null;

        // Act
        void ExecuteInScope()
        {
            using var conn = new DatabaseConnection();
            connection = conn;
            conn.Open();
            conn.IsOpen.Should().BeTrue();
        }

        ExecuteInScope();

        // Assert
        connection.Should().NotBeNull();
        connection!.IsDisposed.Should().BeTrue();
    }
}
```

## Testing Disposal Order

```csharp
public class CompositeResource : IDisposable
{
    private readonly List<IDisposable> _resources;
    private bool _disposed;

    public List<string> DisposalLog { get; } = new();

    public CompositeResource(params IDisposable[] resources)
    {
        _resources = resources.ToList();
    }

    public void Dispose()
    {
        if (_disposed) return;

        // Dispose in reverse order
        for (int i = _resources.Count - 1; i >= 0; i--)
        {
            _resources[i].Dispose();
            DisposalLog.Add($"Disposed resource {i}");
        }

        _disposed = true;
    }
}

[TestClass]
public class DisposalOrderTests
{
    [TestMethod]
    public void Dispose_DisposesInReverseOrder()
    {
        // Arrange
        var resource1 = new TrackingDisposable("Resource1");
        var resource2 = new TrackingDisposable("Resource2");
        var resource3 = new TrackingDisposable("Resource3");

        var composite = new CompositeResource(resource1, resource2, resource3);

        // Act
        composite.Dispose();

        // Assert
        resource3.DisposedBefore(resource2).Should().BeTrue();
        resource2.DisposedBefore(resource1).Should().BeTrue();
    }
}

public class TrackingDisposable : IDisposable
{
    private static int _disposeOrder;
    private int _myDisposeOrder;

    public string Name { get; }
    public bool IsDisposed => _myDisposeOrder > 0;

    public TrackingDisposable(string name) => Name = name;

    public void Dispose()
    {
        if (!IsDisposed)
        {
            _myDisposeOrder = Interlocked.Increment(ref _disposeOrder);
        }
    }

    public bool DisposedBefore(TrackingDisposable other) =>
        IsDisposed && other.IsDisposed && _myDisposeOrder < other._myDisposeOrder;
}
```

## Testing Exception During Disposal

```csharp
public class FaultyResource : IDisposable
{
    private readonly bool _throwOnDispose;

    public bool DisposeCalled { get; private set; }

    public FaultyResource(bool throwOnDispose = false)
    {
        _throwOnDispose = throwOnDispose;
    }

    public void Dispose()
    {
        DisposeCalled = true;
        if (_throwOnDispose)
        {
            throw new InvalidOperationException("Disposal failed");
        }
    }
}

[TestClass]
public class DisposalExceptionTests
{
    [TestMethod]
    public void Dispose_WhenThrows_StillMarksDisposeCalled()
    {
        // Arrange
        var resource = new FaultyResource(throwOnDispose: true);

        // Act
        Action act = () => resource.Dispose();

        // Assert
        act.Should().Throw<InvalidOperationException>()
            .WithMessage("Disposal failed");
        resource.DisposeCalled.Should().BeTrue();
    }

    [TestMethod]
    public void UsingStatement_WhenDisposalThrows_PropagatesException()
    {
        // Arrange & Act
        Action act = () =>
        {
            using var resource = new FaultyResource(throwOnDispose: true);
            // Do work
        };

        // Assert
        act.Should().Throw<InvalidOperationException>();
    }
}
```

## Testing IAsyncDisposable

```csharp
public class AsyncResource : IAsyncDisposable
{
    private bool _disposed;

    public bool IsDisposed => _disposed;

    public async ValueTask DisposeAsync()
    {
        if (!_disposed)
        {
            await CleanupAsync();
            _disposed = true;
        }
    }

    private async Task CleanupAsync()
    {
        await Task.Delay(10); // Simulate async cleanup
    }
}

[TestClass]
public class AsyncDisposableTests
{
    [TestMethod]
    public async Task DisposeAsync_DisposesResource()
    {
        // Arrange
        var resource = new AsyncResource();

        // Act
        await resource.DisposeAsync();

        // Assert
        resource.IsDisposed.Should().BeTrue();
    }

    [TestMethod]
    public async Task AwaitUsingStatement_DisposesAsynchronously()
    {
        // Arrange
        AsyncResource? resource = null;

        // Act
        await using (resource = new AsyncResource())
        {
            resource.IsDisposed.Should().BeFalse();
        }

        // Assert
        resource.IsDisposed.Should().BeTrue();
    }

    [TestMethod]
    public async Task AwaitUsingDeclaration_DisposesAtScopeEnd()
    {
        // Arrange
        AsyncResource? resource = null;

        // Act
        async Task ExecuteInScope()
        {
            await using var res = new AsyncResource();
            resource = res;
            res.IsDisposed.Should().BeFalse();
        }

        await ExecuteInScope();

        // Assert
        resource.Should().NotBeNull();
        resource!.IsDisposed.Should().BeTrue();
    }
}
```

## Testing Dispose Pattern

```csharp
public class ProperDisposable : IDisposable
{
    private bool _disposed;
    private readonly IDisposable _managedResource;

    public bool IsDisposed => _disposed;

    public ProperDisposable(IDisposable managedResource)
    {
        _managedResource = managedResource;
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                _managedResource?.Dispose();
            }
            _disposed = true;
        }
    }

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }
}

[TestClass]
public class DisposePatternTests
{
    [TestMethod]
    public void Dispose_DisposeManagedResources()
    {
        // Arrange
        var managedResource = new TrackingDisposable("Managed");
        var disposable = new ProperDisposable(managedResource);

        // Act
        disposable.Dispose();

        // Assert
        disposable.IsDisposed.Should().BeTrue();
        managedResource.IsDisposed.Should().BeTrue();
    }

    [TestMethod]
    public void Dispose_CalledTwice_IdempotentBehavior()
    {
        // Arrange
        var managedResource = new TrackingDisposable("Managed");
        var disposable = new ProperDisposable(managedResource);

        // Act
        disposable.Dispose();
        disposable.Dispose(); // Call again

        // Assert
        disposable.IsDisposed.Should().BeTrue();
        // Should not throw or cause issues
    }
}
```

## Testing Resource Leak Prevention

```csharp
public class ResourcePool : IDisposable
{
    private readonly List<IDisposable> _resources = new();
    private bool _disposed;

    public int ActiveResourceCount => _resources.Count(r => r is TrackingDisposable t && !t.IsDisposed);

    public T Acquire<T>() where T : IDisposable, new()
    {
        var resource = new T();
        _resources.Add(resource);
        return resource;
    }

    public void Dispose()
    {
        if (_disposed) return;

        foreach (var resource in _resources)
        {
            resource.Dispose();
        }

        _resources.Clear();
        _disposed = true;
    }
}

[TestClass]
public class ResourceLeakTests
{
    [TestMethod]
    public void Dispose_DisposesAllAcquiredResources()
    {
        // Arrange
        var pool = new ResourcePool();
        var resource1 = pool.Acquire<TrackingDisposable>();
        var resource2 = pool.Acquire<TrackingDisposable>();
        var resource3 = pool.Acquire<TrackingDisposable>();

        // Act
        pool.Dispose();

        // Assert
        resource1.IsDisposed.Should().BeTrue();
        resource2.IsDisposed.Should().BeTrue();
        resource3.IsDisposed.Should().BeTrue();
        pool.ActiveResourceCount.Should().Be(0);
    }

    [TestMethod]
    public void PoolNotDisposed_LeavesResourcesActive()
    {
        // Arrange
        var pool = new ResourcePool();
        pool.Acquire<TrackingDisposable>();
        pool.Acquire<TrackingDisposable>();

        // Act - Don't dispose

        // Assert
        pool.ActiveResourceCount.Should().Be(2);
    }
}
```

## Testing SafeHandle and Unmanaged Resources

```csharp
public class UnmanagedResourceWrapper : IDisposable
{
    private IntPtr _handle;
    private bool _disposed;

    public bool HasHandle => _handle != IntPtr.Zero;
    public bool IsDisposed => _disposed;

    public UnmanagedResourceWrapper()
    {
        _handle = AllocateHandle();
    }

    private IntPtr AllocateHandle() => new IntPtr(42); // Simulate allocation

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (_handle != IntPtr.Zero)
            {
                // Free unmanaged resource
                _handle = IntPtr.Zero;
            }
            _disposed = true;
        }
    }

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }

    ~UnmanagedResourceWrapper()
    {
        Dispose(disposing: false);
    }
}

[TestClass]
public class UnmanagedResourceTests
{
    [TestMethod]
    public void Dispose_FreesUnmanagedResource()
    {
        // Arrange
        var wrapper = new UnmanagedResourceWrapper();
        wrapper.HasHandle.Should().BeTrue();

        // Act
        wrapper.Dispose();

        // Assert
        wrapper.IsDisposed.Should().BeTrue();
        wrapper.HasHandle.Should().BeFalse();
    }
}
```

## Why It's Important

1. **Resource Leaks**: Prevent memory and handle leaks
2. **Correctness**: Ensure proper cleanup
3. **Reliability**: Guarantee disposal even with exceptions
4. **Performance**: Release resources promptly
5. **Best Practices**: Follow dispose pattern correctly

## Guidelines

**Testing Disposal**
- Verify `IsDisposed` or equivalent flag
- Test using statements and declarations
- Test disposal order for composite resources
- Verify disposal with exceptions
- Test async disposal with `IAsyncDisposable`

**Disposal Pattern**
- Test idempotent disposal (safe to call twice)
- Verify managed resources are disposed
- Test finalizer doesn't run after Dispose
- Ensure GC.SuppressFinalize is called

**Common Assertions**
- `.IsDisposed.Should().BeTrue()` - verify disposed
- `.Should().Throw<ObjectDisposedException>()` - use after dispose
- Test disposal in using blocks
- Verify resource cleanup order

## Common Pitfalls

❌ **Not testing actual disposal**
```csharp
// Missing disposal verification
using var resource = new MyResource();
// No assertion that it's actually disposed
```

✅ **Verify disposal**
```csharp
// Verify disposal happened
MyResource? resource = null;
using (resource = new MyResource()) { }
resource.IsDisposed.Should().BeTrue();
```

❌ **Not testing exception safety**
```csharp
// What if exception during use?
using var resource = new MyResource();
resource.DoWork(); // Might throw
```

✅ **Test disposal with exceptions**
```csharp
// Ensure disposal even with exception
MyResource? resource = null;
Action act = () =>
{
    using (resource = new MyResource())
    {
        throw new Exception();
    }
};
act.Should().Throw<Exception>();
resource!.IsDisposed.Should().BeTrue();
```

## See Also

- [Lifecycle Management](../patterns/lifecycle-management.md) — object lifecycle
- [Resource Lifetime](../patterns/type-safe-resource-lifetime.md) — type-safe resources
- [Testing Exceptions](./testing-exceptions.md) — exception testing
- [Testing Async Code](./testing-async-code.md) — async disposal
