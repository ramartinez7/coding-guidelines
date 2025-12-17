# Testing Weak References

> Test memory management with weak references—verify objects can be collected when only weakly referenced.

## Problem

Weak references don't prevent garbage collection. Tests must verify weak references allow collection while strong references prevent it.

## Example

```csharp
public class Cache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, WeakReference<TValue>> cache = new();
    
    public void Add(TKey key, TValue value)
    {
        cache[key] = new WeakReference<TValue>(value);
    }
    
    public bool TryGet(TKey key, out TValue? value)
    {
        if (cache.TryGetValue(key, out var weakRef))
        {
            return weakRef.TryGetTarget(out value);
        }
        value = default;
        return false;
    }
}

[Fact]
public void WeakReference_WithoutStrongReference_AllowsCollection()
{
    var cache = new Cache<string, LargeObject>();
    var key = "test";
    
    // Create object and add to cache (only weak reference)
    cache.Add(key, new LargeObject());
    
    // Force garbage collection
    GC.Collect();
    GC.WaitForPendingFinalizers();
    GC.Collect();
    
    // Object should be collected
    var found = cache.TryGet(key, out var value);
    found.Should().BeFalse();
}

[Fact]
public void WeakReference_WithStrongReference_PreventsCollection()
{
    var cache = new Cache<string, LargeObject>();
    var key = "test";
    var strongRef = new LargeObject();
    
    cache.Add(key, strongRef);
    
    GC.Collect();
    GC.WaitForPendingFinalizers();
    GC.Collect();
    
    var found = cache.TryGet(key, out var value);
    found.Should().BeTrue();
    value.Should().BeSameAs(strongRef);
}
```

## See Also

- [Testing Disposable Resources](./testing-disposable-resources.md)
- [Testing Finalizers](./testing-finalizers-destructors.md)
