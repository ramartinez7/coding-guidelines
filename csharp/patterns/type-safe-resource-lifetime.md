# Type-Safe Resource Lifetime (RAII in C#)

> Resources cleaned up with `try-finally` or forgotten `Dispose()` calls leak—use the type system to enforce Resource Acquisition Is Initialization (RAII) and make resource leaks impossible.

## Problem

Manual resource management with `IDisposable` relies on developers remembering to call `Dispose()` or use `using` statements. Resources can leak when exceptions occur, when developers forget disposal, or when ownership transfer is unclear. The compiler doesn't enforce cleanup.

## Example

### ❌ Before

```csharp
public class DataProcessor
{
    public void ProcessFile(string path)
    {
        var file = File.Open(path, FileMode.Open);
        // Forgot using statement—file handle leaks on exception
        
        var data = ReadData(file);
        ProcessData(data);
        
        file.Dispose();  // Never reached if ProcessData throws
    }
    
    public FileStream GetFileHandle(string path)
    {
        // Returns resource—caller must remember to dispose
        return File.Open(path, FileMode.Open);
    }
    
    public void UseMultipleResources()
    {
        var file1 = File.Open("data1.txt", FileMode.Open);
        var file2 = File.Open("data2.txt", FileMode.Open);
        // If file2 throws, file1 leaks
        
        ProcessBoth(file1, file2);
        
        file1.Dispose();
        file2.Dispose();
    }
}

// Problems:
// - Resources leak on exceptions
// - Caller forgets to dispose
// - Multiple resources need nested try-finally
// - Ownership unclear
```

### ✅ After

```csharp
/// <summary>
/// Owned resource that must be disposed—cannot escape scope.
/// </summary>
public readonly ref struct OwnedResource<T> where T : IDisposable
{
    private readonly T _resource;
    
    public OwnedResource(T resource)
    {
        _resource = resource;
    }
    
    public T Value => _resource;
    
    public void Dispose()
    {
        _resource.Dispose();
    }
    
    // Use the resource within a scope
    public TResult Use<TResult>(Func<T, TResult> action)
    {
        try
        {
            return action(_resource);
        }
        finally
        {
            _resource.Dispose();
        }
    }
    
    public void Use(Action<T> action)
    {
        try
        {
            action(_resource);
        }
        finally
        {
            _resource.Dispose();
        }
    }
}

/// <summary>
/// Borrowed resource—doesn't own, so can't dispose.
/// </summary>
public readonly ref struct BorrowedResource<T>
{
    private readonly T _resource;
    
    public BorrowedResource(T resource)
    {
        _resource = resource;
    }
    
    public T Value => _resource;
    
    // No Dispose—doesn't own the resource
}

public static class ResourceExtensions
{
    public static OwnedResource<T> Owned<T>(this T resource) where T : IDisposable
        => new(resource);
    
    public static BorrowedResource<T> Borrowed<T>(this T resource)
        => new(resource);
}

// Usage: Compiler enforces cleanup
public class DataProcessor
{
    public void ProcessFile(string path)
    {
        // ref struct prevents escaping—must be disposed before return
        using var file = File.Open(path, FileMode.Open).Owned();
        
        var data = ReadData(file.Value);
        ProcessData(data);
        
        // Automatic disposal even on exception
    }
    
    public string ReadAllText(string path)
    {
        return File.Open(path, FileMode.Open)
            .Owned()
            .Use(file =>
            {
                using var reader = new StreamReader(file);
                return reader.ReadToEnd();
            });
        // File automatically disposed after Use
    }
    
    public void UseMultipleResources()
    {
        using var file1 = File.Open("data1.txt", FileMode.Open).Owned();
        using var file2 = File.Open("data2.txt", FileMode.Open).Owned();
        
        ProcessBoth(file1.Value, file2.Value);
        
        // Both disposed in reverse order automatically
    }
    
    // Borrowed resources make ownership explicit
    private void ProcessBorrowed(BorrowedResource<FileStream> file)
    {
        // Can use but not dispose
        var data = ReadData(file.Value);
        // Caller owns disposal
    }
}

// Won't compile:
// OwnedResource<FileStream> GetFile() => File.Open("x").Owned();
// Error: ref struct cannot be returned
```

## Scoped Resource with Phantom Types

```csharp
public interface IResourceState { }
public interface IAcquired : IResourceState { }
public interface IReleased : IResourceState { }

public readonly ref struct ScopedResource<T, TState> 
    where T : IDisposable
    where TState : IResourceState
{
    private readonly T _resource;
    
    private ScopedResource(T resource)
    {
        _resource = resource;
    }
    
    public static ScopedResource<T, IAcquired> Acquire(T resource)
        => new(resource);
    
    public T Value => _resource;
}

public static class ScopedResourceExtensions
{
    public static ScopedResource<T, IReleased> Release<T>(
        this ScopedResource<T, IAcquired> resource)
        where T : IDisposable
    {
        resource.Value.Dispose();
        return new ScopedResource<T, IReleased>(default(T)!);
    }
    
    public static TResult Use<T, TResult>(
        this ScopedResource<T, IAcquired> resource,
        Func<T, TResult> action)
        where T : IDisposable
    {
        try
        {
            return action(resource.Value);
        }
        finally
        {
            resource.Value.Dispose();
        }
    }
}

// Usage: State transitions tracked in types
using var acquired = ScopedResource<FileStream, IAcquired>.Acquire(
    File.Open("data.txt", FileMode.Open));

var content = acquired.Use(file =>
{
    using var reader = new StreamReader(file);
    return reader.ReadToEnd();
});

// Won't compile:
// acquired.Value.Dispose();  // Can't dispose directly
// var released = acquired.Release().Value;  // Can't use after release
```

## Database Transaction Lifetime

```csharp
public readonly ref struct TransactionScope
{
    private readonly IDbTransaction _transaction;
    
    public TransactionScope(IDbConnection connection)
    {
        _transaction = connection.BeginTransaction();
    }
    
    public IDbTransaction Transaction => _transaction;
    
    public void Commit()
    {
        _transaction.Commit();
    }
    
    public void Dispose()
    {
        // Rollback if not committed
        if (_transaction.Connection is not null)
            _transaction.Rollback();
        
        _transaction.Dispose();
    }
}

// Usage: Transaction auto-rolls back if not committed
public void TransferFunds(AccountId from, AccountId to, Money amount)
{
    using var tx = new TransactionScope(_connection);
    
    Debit(from, amount, tx.Transaction);
    Credit(to, amount, tx.Transaction);
    
    tx.Commit();
    
    // Auto-rollback if Commit not reached
}
```

## Async Resource Management

```csharp
public sealed class AsyncOwnedResource<T> : IAsyncDisposable where T : IAsyncDisposable
{
    private readonly T _resource;
    private bool _disposed;
    
    private AsyncOwnedResource(T resource)
    {
        _resource = resource;
    }
    
    public static AsyncOwnedResource<T> Acquire(T resource)
        => new(resource);
    
    public T Value
    {
        get
        {
            if (_disposed)
                throw new ObjectDisposedException(nameof(AsyncOwnedResource<T>));
            return _resource;
        }
    }
    
    public async ValueTask<TResult> UseAsync<TResult>(
        Func<T, ValueTask<TResult>> action)
    {
        try
        {
            return await action(_resource);
        }
        finally
        {
            await DisposeAsync();
        }
    }
    
    public async ValueTask DisposeAsync()
    {
        if (!_disposed)
        {
            await _resource.DisposeAsync();
            _disposed = true;
        }
    }
}

// Usage: Async resource safety
public async Task<string> ReadFileAsync(string path)
{
    var stream = File.OpenRead(path);
    var resource = AsyncOwnedResource<FileStream>.Acquire(stream);
    
    return await resource.UseAsync(async file =>
    {
        using var reader = new StreamReader(file);
        return await reader.ReadToEndAsync();
    });
    // Automatically disposed
}
```

## Why It's a Problem

1. **Resource leaks**: Forgotten disposal or exceptions prevent cleanup
2. **Unclear ownership**: Who is responsible for disposing?
3. **Multiple resources**: Nested try-finally blocks are error-prone
4. **No compiler enforcement**: Nothing prevents escaping undisposed resources

## Symptoms

- `using` statements scattered throughout code
- Try-finally blocks for cleanup
- Resource leak bugs in production
- Comments like "remember to dispose"
- Tests checking for proper disposal

## Benefits

- **Automatic cleanup**: Resources disposed automatically
- **Exception safety**: Cleanup happens even on exceptions
- **Clear ownership**: Types make ownership explicit
- **Compiler enforcement**: ref struct prevents escaping
- **Composition**: Multiple resources compose cleanly

## See Also

- [Phantom Types](./phantom-types.md) — state tracking in types
- [Type-Safe State Transitions](./type-safe-state-transitions.md) — state machines
- [Memory Safety (Span)](./memory-safety-span.md) — zero-allocation patterns
- [Async Patterns](./async-patterns.md) — async resource management
