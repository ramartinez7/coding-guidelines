# Resource Management Best Practices

> Properly manage resources with using statements, IDisposable pattern, and async disposal to prevent leaks.

## Problem

Improper resource management leads to file handles, database connections, and memory leaks that degrade performance and cause application failures.

## Example

### ❌ Before

```csharp
public class FileProcessor
{
    public void ProcessFile(string path)
    {
        var stream = File.OpenRead(path);
        var reader = new StreamReader(stream);
        var content = reader.ReadToEnd();
        ProcessContent(content);
        // Resources never disposed!
    }

    public async Task DownloadFileAsync(string url)
    {
        var client = new HttpClient();
        var response = await client.GetAsync(url);
        var content = await response.Content.ReadAsStringAsync();
        SaveContent(content);
        // HttpClient not disposed, socket exhaustion!
    }
}
```

### ✅ After

```csharp
public class FileProcessor
{
    public void ProcessFile(string path)
    {
        using var stream = File.OpenRead(path);
        using var reader = new StreamReader(stream);
        var content = reader.ReadToEnd();
        ProcessContent(content);
    }  // Automatic disposal

    private static readonly HttpClient HttpClient = new();

    public async Task DownloadFileAsync(string url)
    {
        using var response = await HttpClient.GetAsync(url);
        var content = await response.Content.ReadAsStringAsync();
        SaveContent(content);
    }
}
```

## Best Practices

### 1. Use 'using' Declarations (C# 8+)

```csharp
// ❌ Using statement (extra indentation)
public void OldStyle(string path)
{
    using (var stream = File.OpenRead(path))
    {
        using (var reader = new StreamReader(stream))
        {
            var content = reader.ReadToEnd();
            Process(content);
        }
    }
}

// ✅ Using declaration (cleaner)
public void NewStyle(string path)
{
    using var stream = File.OpenRead(path);
    using var reader = new StreamReader(stream);
    var content = reader.ReadToEnd();
    Process(content);
}  // Disposed at end of method
```

### 2. Implement IDisposable Correctly

```csharp
// ✅ Proper IDisposable implementation
public sealed class ResourceHolder : IDisposable
{
    private FileStream? fileStream;
    private bool disposed;

    public ResourceHolder(string path)
    {
        this.fileStream = File.OpenRead(path);
    }

    public void Dispose()
    {
        if (disposed)
        {
            return;
        }

        fileStream?.Dispose();
        fileStream = null;
        disposed = true;
    }
}

// ✅ For inheritance scenarios
public class BaseResource : IDisposable
{
    private FileStream? fileStream;
    private bool disposed;

    protected virtual void Dispose(bool disposing)
    {
        if (disposed)
        {
            return;
        }

        if (disposing)
        {
            // Dispose managed resources
            fileStream?.Dispose();
            fileStream = null;
        }

        // Free unmanaged resources here if any

        disposed = true;
    }

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }

    ~BaseResource()
    {
        Dispose(disposing: false);
    }
}
```

### 3. Use IAsyncDisposable for Async Resources

```csharp
// ✅ Async disposal
public sealed class AsyncResourceHolder : IAsyncDisposable
{
    private Stream? stream;

    public async ValueTask DisposeAsync()
    {
        if (stream != null)
        {
            await stream.DisposeAsync();
            stream = null;
        }
    }
}

// Usage
public async Task ProcessAsync()
{
    await using var resource = new AsyncResourceHolder();
    await resource.ProcessDataAsync();
}  // Async disposal
```

### 4. Don't Create HttpClient Per Request

```csharp
// ❌ Socket exhaustion
public class BadHttpService
{
    public async Task<string> GetDataAsync(string url)
    {
        using var client = new HttpClient();  // DON'T DO THIS!
        return await client.GetStringAsync(url);
    }
}

// ✅ Reuse HttpClient
public class GoodHttpService
{
    private static readonly HttpClient HttpClient = new();

    public async Task<string> GetDataAsync(string url)
    {
        return await HttpClient.GetStringAsync(url);
    }
}

// ✅ Or use IHttpClientFactory
public class BestHttpService
{
    private readonly IHttpClientFactory clientFactory;

    public BestHttpService(IHttpClientFactory clientFactory)
    {
        this.clientFactory = clientFactory;
    }

    public async Task<string> GetDataAsync(string url)
    {
        using var client = clientFactory.CreateClient();
        return await client.GetStringAsync(url);
    }
}
```

### 5. Use ObjectPool for Frequently Created Objects

```csharp
// ❌ Constantly creating and disposing
public class Processor
{
    public void ProcessMany(List<Item> items)
    {
        foreach (var item in items)
        {
            using var buffer = new MemoryStream();
            ProcessItem(item, buffer);
        }  // Creates lots of GC pressure
    }
}

// ✅ Use ObjectPool
public class Processor
{
    private readonly ObjectPool<MemoryStream> streamPool;

    public Processor()
    {
        var policy = new MemoryStreamPooledObjectPolicy();
        this.streamPool = new DefaultObjectPool<MemoryStream>(policy);
    }

    public void ProcessMany(List<Item> items)
    {
        foreach (var item in items)
        {
            var buffer = streamPool.Get();
            try
            {
                ProcessItem(item, buffer);
            }
            finally
            {
                buffer.SetLength(0);  // Clear for reuse
                streamPool.Return(buffer);
            }
        }
    }
}
```

### 6. Dispose in Finally Block If Not Using 'using'

```csharp
// ❌ Resource leak on exception
public void Process()
{
    var connection = OpenConnection();
    DoWork(connection);
    connection.Dispose();  // Never called if DoWork throws!
}

// ✅ Dispose in finally
public void Process()
{
    DatabaseConnection? connection = null;
    try
    {
        connection = OpenConnection();
        DoWork(connection);
    }
    finally
    {
        connection?.Dispose();
    }
}

// ✅ Better: use 'using'
public void Process()
{
    using var connection = OpenConnection();
    DoWork(connection);
}
```

### 7. Use ArrayPool for Large Buffers

```csharp
// ❌ Allocating large arrays
public byte[] ProcessData(Stream input)
{
    var buffer = new byte[4096];  // GC pressure
    input.Read(buffer, 0, buffer.Length);
    return Process(buffer);
}

// ✅ Use ArrayPool
public byte[] ProcessData(Stream input)
{
    var buffer = ArrayPool<byte>.Shared.Rent(4096);
    try
    {
        input.Read(buffer, 0, 4096);
        return Process(buffer.AsSpan(0, 4096));
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

### 8. Avoid Capturing Disposables in Closures

```csharp
// ❌ Stream disposed before async operation completes
public Task<string> ReadFileAsync(string path)
{
    using var stream = File.OpenRead(path);
    return Task.Run(() =>
    {
        using var reader = new StreamReader(stream);  // stream already disposed!
        return reader.ReadToEnd();
    });
}

// ✅ Keep resource alive for entire async operation
public async Task<string> ReadFileAsync(string path)
{
    using var stream = File.OpenRead(path);
    using var reader = new StreamReader(stream);
    return await Task.Run(() => reader.ReadToEnd());
}
```

### 9. Use Weak References for Caches

```csharp
// ❌ Strong references prevent GC
public class Cache
{
    private readonly Dictionary<string, LargeObject> cache = new();

    public LargeObject Get(string key)
    {
        return cache[key];  // Keeps objects alive forever!
    }
}

// ✅ WeakReference allows GC when memory pressure
public class Cache
{
    private readonly Dictionary<string, WeakReference<LargeObject>> cache = new();

    public bool TryGet(string key, out LargeObject? value)
    {
        value = null;
        return cache.TryGetValue(key, out var weakRef) 
            && weakRef.TryGetTarget(out value);
    }

    public void Set(string key, LargeObject value)
    {
        cache[key] = new WeakReference<LargeObject>(value);
    }
}

// ✅ Or use MemoryCache with eviction policies
private readonly MemoryCache cache = new(new MemoryCacheOptions
{
    SizeLimit = 1024,
    ExpirationScanFrequency = TimeSpan.FromMinutes(5)
});
```

### 10. Be Careful with IEnumerable and Dispose

```csharp
// ❌ Dispose called before enumeration
public IEnumerable<string> ReadLines(string path)
{
    using var reader = File.OpenText(path);
    return reader.ReadToEnd().Split('\n');  // OK, materialized
}

// ❌ Iterator disposed too early
public IEnumerable<string> ReadLines(string path)
{
    using var reader = File.OpenText(path);  // Disposed before enumeration!
    while (!reader.EndOfStream)
    {
        yield return reader.ReadLine()!;
    }
}

// ✅ Caller disposes
public IEnumerable<string> ReadLines(string path)
{
    var reader = File.OpenText(path);
    try
    {
        while (!reader.EndOfStream)
        {
            yield return reader.ReadLine()!;
        }
    }
    finally
    {
        reader?.Dispose();
    }
}
```

### 11. Use Scoped Services in DI

```csharp
// ❌ Singleton with disposable dependency
services.AddSingleton<IOrderService, OrderService>();  // Leaks DbContext!

public class OrderService : IOrderService
{
    private readonly DbContext context;

    public OrderService(DbContext context)  // DbContext is scoped!
    {
        this.context = context;
    }
}

// ✅ Match lifetime appropriately
services.AddScoped<IOrderService, OrderService>();
services.AddScoped<DbContext>();
```

### 12. Implement IDisposable with Dependency Injection

```csharp
// ✅ DI container handles disposal
public class OrderProcessor : IDisposable
{
    private readonly DbContext context;
    private readonly FileStream logStream;

    public OrderProcessor(DbContext context)
    {
        this.context = context;
        this.logStream = File.OpenWrite("orders.log");
    }

    public void Dispose()
    {
        // Don't dispose DI-provided dependencies!
        // context is disposed by DI container

        // Only dispose resources we created
        logStream?.Dispose();
    }
}
```

## Symptoms

- OutOfMemoryException from resource leaks
- "Too many open files" errors
- Socket exhaustion
- Database connection pool exhaustion
- Poor performance from excessive GC

## Benefits

- **No resource leaks** with proper disposal
- **Better performance** with resource reuse
- **Predictable behavior** with deterministic cleanup
- **Scalability** by not exhausting system resources
- **Maintainability** with clear resource ownership

## See Also

- [Type-Safe Resource Lifetime](./type-safe-resource-lifetime.md) — RAII pattern
- [Allocation Budget](./allocation-budget.md) — Reducing allocations
- [Memory Safety](./memory-safety-span.md) — Span<T> for stack allocation
