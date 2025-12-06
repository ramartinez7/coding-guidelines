# Allocation Budget (Zero-Allocation Patterns)

> High-throughput code allocates excessively, triggering frequent GC pauses—use pooling, stack allocation, and budget tracking to eliminate allocations in hot paths.

## Problem

Every allocation eventually needs garbage collection. In high-throughput scenarios (100k+ ops/sec), allocations accumulate faster than the GC can clean them, causing unpredictable latency spikes. Code that "feels fast" locally can fail under production load.

## Example

### ❌ Before

```csharp
public class RequestHandler
{
    public Response HandleRequest(string requestData)
    {
        // Allocation 1: Parse creates a new object
        var request = JsonSerializer.Deserialize<Request>(requestData);
        
        // Allocation 2: ToUpper creates a new string
        var normalizedPath = request.Path.ToUpper();
        
        // Allocation 3: Split creates a string array
        var segments = normalizedPath.Split('/');
        
        // Allocation 4: LINQ Select creates an IEnumerable
        var results = segments.Select(s => ProcessSegment(s)).ToList();
        
        // Allocation 5: New response object
        return new Response 
        { 
            Results = results,
            ProcessedAt = DateTime.UtcNow 
        };
    }
    
    private string ProcessSegment(string segment)
    {
        // Allocation 6: String interpolation
        return $"Processed: {segment}";
    }
}

// At 100k requests/sec, this creates 600k+ allocations/sec
// GC Gen0 collections happen every 10-20ms → latency spikes
```

**Problems:**
- 6+ allocations per request
- GC pauses every few milliseconds
- Unpredictable p99 latencies
- Memory bandwidth wasted

### ✅ After

```csharp
public class RequestHandler
{
    // Object pool for request/response objects
    private readonly ObjectPool<Request> _requestPool;
    private readonly ObjectPool<Response> _responsePool;
    private readonly ObjectPool<List<string>> _listPool;
    
    public Response HandleRequest(ReadOnlySpan<char> requestData)
    {
        // Rent from pool (zero allocation if pool has capacity)
        var request = _requestPool.Get();
        var response = _responsePool.Get();
        var results = _listPool.Get();
        
        try
        {
            // Parse into pooled object (allocation-free with Utf8JsonReader)
            ParseRequest(requestData, request);
            
            // Stack-allocated span for path normalization (zero allocation)
            Span<char> normalizedPath = stackalloc char[request.Path.Length];
            request.Path.AsSpan().ToUpperInvariant(normalizedPath);
            
            // Split without allocation using Span
            foreach (var segment in normalizedPath.Split('/'))
            {
                // Stack-allocated buffer for result (zero allocation)
                Span<char> resultBuffer = stackalloc char[256];
                int written = ProcessSegment(segment, resultBuffer);
                results.Add(new string(resultBuffer[..written]));
            }
            
            response.Results = results;
            response.ProcessedAt = DateTime.UtcNow;
            
            return response;
        }
        finally
        {
            // Return to pool for reuse
            _requestPool.Return(request);
            // Note: response is returned by caller after use
        }
    }
    
    private int ProcessSegment(ReadOnlySpan<char> segment, Span<char> destination)
    {
        // Write directly to destination buffer (zero allocation)
        "Processed: ".AsSpan().CopyTo(destination);
        segment.CopyTo(destination[11..]);
        return 11 + segment.Length;
    }
}

// At 100k requests/sec, allocations drop to near-zero
// GC Gen0 collections happen every few seconds instead of milliseconds
```

## Object Pooling

```csharp
public class ObjectPool<T> where T : class, new()
{
    private readonly ConcurrentBag<T> _objects = new();
    private readonly Func<T> _objectGenerator;
    private readonly int _maxPoolSize;
    
    public ObjectPool(Func<T>? objectGenerator = null, int maxPoolSize = 100)
    {
        _objectGenerator = objectGenerator ?? (() => new T());
        _maxPoolSize = maxPoolSize;
    }
    
    public T Get()
    {
        return _objects.TryTake(out var item) ? item : _objectGenerator();
    }
    
    public void Return(T item)
    {
        if (_objects.Count < _maxPoolSize)
        {
            // Reset state before returning to pool
            if (item is IResettable resettable)
                resettable.Reset();
            
            _objects.Add(item);
        }
        // If pool is full, let it be garbage collected
    }
}

public interface IResettable
{
    void Reset();
}

public class Request : IResettable
{
    public string Path { get; set; } = string.Empty;
    public Dictionary<string, string> Headers { get; set; } = new();
    public byte[] Body { get; set; } = Array.Empty<byte>();
    
    public void Reset()
    {
        Path = string.Empty;
        Headers.Clear();
        Body = Array.Empty<byte>();
    }
}

// Usage with using pattern
public readonly struct PooledObject<T> : IDisposable where T : class
{
    private readonly ObjectPool<T> _pool;
    public T Object { get; }
    
    public PooledObject(ObjectPool<T> pool, T obj)
    {
        _pool = pool;
        Object = obj;
    }
    
    public void Dispose() => _pool.Return(Object);
}

// Convenient usage
using var request = new PooledObject<Request>(_requestPool, _requestPool.Get());
ProcessRequest(request.Object);
// Automatically returned to pool on scope exit
```

## Array Pooling

```csharp
public class BufferHandler
{
    private readonly ArrayPool<byte> _bufferPool = ArrayPool<byte>.Shared;
    
    public void ProcessData(ReadOnlySpan<byte> input)
    {
        // Rent array from shared pool
        byte[] buffer = _bufferPool.Rent(input.Length * 2);
        
        try
        {
            // Use the buffer
            Span<byte> bufferSpan = buffer.AsSpan(0, input.Length * 2);
            ProcessInto(input, bufferSpan);
            
            // Do something with processed data
            Send(bufferSpan);
        }
        finally
        {
            // Always return to pool
            _bufferPool.Return(buffer);
        }
    }
    
    private void ProcessInto(ReadOnlySpan<byte> input, Span<byte> output)
    {
        // Zero-allocation processing
        for (int i = 0; i < input.Length; i++)
        {
            output[i * 2] = (byte)(input[i] >> 4);
            output[i * 2 + 1] = (byte)(input[i] & 0x0F);
        }
    }
}
```

## Stack Allocation Budget

```csharp
public class StringProcessor
{
    private const int MaxStackAlloc = 512;  // Stay under 1KB stack allocation
    
    public string Normalize(string input)
    {
        // Use stack for small inputs
        if (input.Length <= MaxStackAlloc)
        {
            Span<char> buffer = stackalloc char[input.Length];
            input.AsSpan().ToUpperInvariant(buffer);
            return new string(buffer);
        }
        
        // Use array pool for large inputs
        var arrayPool = ArrayPool<char>.Shared;
        char[] rented = arrayPool.Rent(input.Length);
        try
        {
            Span<char> buffer = rented.AsSpan(0, input.Length);
            input.AsSpan().ToUpperInvariant(buffer);
            return new string(buffer);
        }
        finally
        {
            arrayPool.Return(rented);
        }
    }
}
```

## Allocation Tracking

```csharp
public readonly struct AllocationTracker : IDisposable
{
    private readonly long _startBytes;
    private readonly string _operationName;
    
    public AllocationTracker(string operationName)
    {
        _operationName = operationName;
        _startBytes = GC.GetAllocatedBytesForCurrentThread();
    }
    
    public void Dispose()
    {
        var endBytes = GC.GetAllocatedBytesForCurrentThread();
        var allocated = endBytes - _startBytes;
        
        if (allocated > 0)
        {
            Console.WriteLine($"{_operationName} allocated {allocated:N0} bytes");
        }
    }
}

// Usage
public void ProcessRequest()
{
    using var tracker = new AllocationTracker(nameof(ProcessRequest));
    
    // Your code here
    
    // On exit, prints allocation stats
}
```

## Struct-Based Collections

```csharp
// Instead of List<T> for small collections
public ref struct StackList<T>
{
    private Span<T> _items;
    private int _count;
    
    public StackList(Span<T> buffer)
    {
        _items = buffer;
        _count = 0;
    }
    
    public int Count => _count;
    
    public void Add(T item)
    {
        if (_count >= _items.Length)
            throw new InvalidOperationException("Stack list full");
        
        _items[_count++] = item;
    }
    
    public ReadOnlySpan<T> AsSpan() => _items[.._count];
    
    public ref T this[int index] => ref _items[index];
}

// Usage
Span<int> buffer = stackalloc int[100];
var list = new StackList<int>(buffer);

for (int i = 0; i < 50; i++)
    list.Add(i * 2);

// Zero heap allocations
```

## Benchmark Example

```csharp
[MemoryDiagnoser]
public class AllocationBenchmark
{
    private const string TestData = "GET /api/users/123 HTTP/1.1";
    private readonly ObjectPool<Request> _pool = new();
    
    [Benchmark(Baseline = true)]
    public Request AllocatingVersion()
    {
        var parts = TestData.Split(' ');
        return new Request
        {
            Method = parts[0],
            Path = parts[1],
            Protocol = parts[2]
        };
    }
    
    [Benchmark]
    public Request PooledVersion()
    {
        var request = _pool.Get();
        
        // Parse using Span (zero allocation)
        var span = TestData.AsSpan();
        int firstSpace = span.IndexOf(' ');
        int secondSpace = span[(firstSpace + 1)..].IndexOf(' ') + firstSpace + 1;
        
        request.Method = new string(span[..firstSpace]);
        request.Path = new string(span[(firstSpace + 1)..secondSpace]);
        request.Protocol = new string(span[(secondSpace + 1)..]);
        
        return request;
    }
}

// Results (approximate):
// |           Method |     Mean | Allocated |
// |----------------- |---------:|----------:|
// | AllocatingVersion| 125 ns   |     168 B |
// | PooledVersion    |  45 ns   |       0 B |
```

## Allocation Budget Enforcement

```csharp
public static class AllocationBudget
{
    public static void EnforceZeroAlloc(Action action)
    {
        var before = GC.GetAllocatedBytesForCurrentThread();
        
        action();
        
        var after = GC.GetAllocatedBytesForCurrentThread();
        var allocated = after - before;
        
        if (allocated > 0)
        {
            throw new AllocationBudgetExceededException(
                $"Expected zero allocations, but {allocated} bytes were allocated");
        }
    }
    
    public static void EnforceBudget(int maxBytes, Action action)
    {
        var before = GC.GetAllocatedBytesForCurrentThread();
        
        action();
        
        var after = GC.GetAllocatedBytesForCurrentThread();
        var allocated = after - before;
        
        if (allocated > maxBytes)
        {
            throw new AllocationBudgetExceededException(
                $"Allocated {allocated} bytes, exceeding budget of {maxBytes} bytes");
        }
    }
}

public class AllocationBudgetExceededException : Exception
{
    public AllocationBudgetExceededException(string message) : base(message) { }
}

// Usage in tests
[Fact]
public void ProcessRequest_ShouldNotAllocate()
{
    var handler = new RequestHandler();
    
    AllocationBudget.EnforceZeroAlloc(() =>
    {
        handler.ProcessRequest("test data".AsSpan());
    });
}
```

## Why It's a Problem

1. **GC pauses**: Frequent allocations trigger Gen0 collections every 10-20ms
2. **Unpredictable latency**: p99 and p99.9 latencies spike during GC
3. **Memory bandwidth**: Allocating and collecting wastes CPU and memory bandwidth
4. **Scalability ceiling**: Can't handle more throughput without more GC pauses

## Symptoms

- High Gen0 collection frequency (multiple times per second)
- Latency spikes correlated with GC pauses
- Performance degrades under load
- Memory profiler shows short-lived objects dominating allocations
- P99 latency much higher than P50

## Benefits

- **Predictable latency**: Fewer GC pauses = consistent response times
- **Higher throughput**: Less time spent in GC = more time doing work
- **Better scalability**: Can handle more load without adding hardware
- **Lower memory usage**: Pooled objects reused instead of allocated

## Trade-offs

- **Code complexity**: Pooling and span-based code is more complex
- **Careful state management**: Pooled objects must be reset properly
- **Stack overflow risk**: Too much stack allocation can cause stack overflow
- **Not always worth it**: Only optimize hot paths where allocations matter

## See Also

- [Memory Safety (Span)](./memory-safety-span.md) — zero-allocation string processing
- [Struct Layout](./struct-layout.md) — memory-efficient structs
- [Value Semantics](./value-semantics.md) — stack vs heap allocation
