# Testing Finalizers and Cleanup Patterns

> Test finalizers and cleanup patterns—verify resources are released correctly.

## Problem

Finalizers ensure unmanaged resources are cleaned up. Tests must verify finalization occurs and resources are released.

## Example

```csharp
public class UnmanagedResource : IDisposable
{
    private IntPtr handle;
    private bool disposed;
    
    ~UnmanagedResource()
    {
        Dispose(false);
    }
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (!disposed)
        {
            if (disposing)
            {
                // Dispose managed resources
            }
            
            // Free unmanaged resources
            if (handle != IntPtr.Zero)
            {
                // Free handle
                handle = IntPtr.Zero;
            }
            
            disposed = true;
        }
    }
}

[Fact]
public void Dispose_CallsSuppressFinalize()
{
    var resource = new UnmanagedResource();
    
    resource.Dispose();
    
    // Finalizer should not run
    GC.Collect();
    GC.WaitForPendingFinalizers();
    
    // Verify resource is disposed
    resource.IsDisposed.Should().BeTrue();
}

[Fact]
public void Finalizer_WithoutDispose_ReleasesResources()
{
    var tracker = new FinalizationTracker();
    
    CreateAndAbandonResource(tracker);
    
    GC.Collect();
    GC.WaitForPendingFinalizers();
    GC.Collect();
    
    tracker.FinalizerCalled.Should().BeTrue();
}

private void CreateAndAbandonResource(FinalizationTracker tracker)
{
    var resource = new TrackedResource(tracker);
    // Resource goes out of scope without Dispose
}
```

## Testing SafeHandle Pattern

```csharp
[Fact]
public void SafeHandle_ReleasedProperly()
{
    var handle = new SafeFileHandle();
    
    handle.Dispose();
    
    handle.IsClosed.Should().BeTrue();
    handle.IsInvalid.Should().BeTrue();
}
```

## See Also

- [Testing Disposable Resources](./testing-disposable-resources.md)
- [Testing Weak References](./testing-weak-references.md)
