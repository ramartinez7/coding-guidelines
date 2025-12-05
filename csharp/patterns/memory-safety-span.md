# Memory Safety (Span and Ref Structs)

> String manipulation causes GC pressure—use `Span<T>` and `ref struct` for zero-allocation, memory-safe code enforced by the type system.

## Problem

String manipulation (substrings, parsing, splitting) is a major source of memory allocations. Each operation creates new heap objects that the Garbage Collector must track and clean up. In high-throughput scenarios (100k+ req/sec), this kills performance. C# introduced `Span<T>`, a `ref struct` that the type system *enforces* cannot escape to the heap—preventing use-after-free bugs while enabling zero-allocation code.

## Example

### ❌ Before

```csharp
public static (string Key, string Value) ParseHeader(string header)
{
    // Allocation 1: new string from Substring
    int colonIndex = header.IndexOf(':');
    string key = header.Substring(0, colonIndex);

    // Allocation 2: new string from Substring
    string value = header.Substring(colonIndex + 1);

    // Allocation 3: new string from Trim
    value = value.Trim();

    return (key, value);
}

// Processing 100k headers = 300k+ allocations
foreach (var header in headers)
{
    var (key, value) = ParseHeader(header);
    ProcessHeader(key, value);
}
```

**Hidden costs:**
- 3+ heap allocations per call
- GC pressure in high-throughput scenarios
- Each substring copies bytes to a new location

### ✅ After

```csharp
public static (ReadOnlySpan<char> Key, ReadOnlySpan<char> Value) ParseHeader(
    ReadOnlySpan<char> header)
{
    // Zero allocations: just pointer arithmetic
    int colonIndex = header.IndexOf(':');
    ReadOnlySpan<char> key = header[..colonIndex];
    ReadOnlySpan<char> value = header[(colonIndex + 1)..].Trim();

    return (key, value);
}

// Processing 100k headers = 0 allocations
foreach (var header in headers)
{
    var (key, value) = ParseHeader(header.AsSpan());
    ProcessHeader(key, value);
}
```

**What `Span<T>` represents:**

```
Original string: "Content-Type: application/json"
                  ^            ^                 ^
                  |            |                 |
                  ptr=0        ptr=13            ptr=30

key:   Span<char> { ptr=0,  length=12 }  → "Content-Type"
value: Span<char> { ptr=14, length=16 }  → "application/json"

No copying—just different windows into the same memory.
```

## The Ref Struct Constraint

`Span<T>` is a `ref struct`—the compiler enforces it cannot escape to the heap:

```csharp
public ref struct Span<T>
{
    private readonly ref T _pointer;
    private readonly int _length;
}

// ❌ Compile error: cannot store ref struct in a class field
public class Parser
{
    private Span<char> _buffer;  // Error: cannot have ref struct as field
}

// ❌ Compile error: cannot box a ref struct
object boxed = someSpan;  // Error

// ❌ Compile error: cannot use in async methods
async Task ProcessAsync(Span<char> data)  // Error
{
    await Task.Delay(100);
    // Span could outlive the stack frame
}

// ❌ Compile error: cannot capture in lambda
Span<char> data = stackalloc char[100];
var func = () => data.Length;  // Error: cannot capture ref struct
```

**Why this matters:** These compile-time restrictions prevent use-after-free bugs. The span cannot outlive the memory it points to.

## Common Span Operations

### Parsing Without Allocation

```csharp
public static bool TryParseQueryString(
    ReadOnlySpan<char> query,
    out ReadOnlySpan<char> key,
    out ReadOnlySpan<char> value)
{
    int equalsIndex = query.IndexOf('=');
    if (equalsIndex < 0)
    {
        key = default;
        value = default;
        return false;
    }

    key = query[..equalsIndex];
    value = query[(equalsIndex + 1)..];
    return true;
}

// Usage
ReadOnlySpan<char> query = "name=John&age=30".AsSpan();
foreach (var segment in query.Split('&'))
{
    if (TryParseQueryString(segment, out var key, out var value))
    {
        Console.WriteLine($"{key}: {value}");
    }
}
```

### Stack Allocation

```csharp
public static string FormatId(Guid id)
{
    // Allocate 36 chars on the stack, not the heap
    Span<char> buffer = stackalloc char[36];

    id.TryFormat(buffer, out int charsWritten);

    return new string(buffer[..charsWritten]);  // Only one allocation: the final string
}

// Compare to the allocating version:
public static string FormatIdAllocating(Guid id)
{
    return id.ToString();  // Allocates a new string
}
```

### Span-Based String Building

```csharp
public static string BuildGreeting(ReadOnlySpan<char> name)
{
    // Fixed-size buffer on stack
    Span<char> buffer = stackalloc char[256];
    int position = 0;

    "Hello, ".AsSpan().CopyTo(buffer[position..]);
    position += 7;

    name.CopyTo(buffer[position..]);
    position += name.Length;

    "!".AsSpan().CopyTo(buffer[position..]);
    position += 1;

    return new string(buffer[..position]);
}
```

## Memory<T> for Async Scenarios

When you need heap storage but span-like slicing:

```csharp
public class BufferPool
{
    private readonly Memory<byte> _buffer;

    public BufferPool(int size)
    {
        _buffer = new byte[size];  // Heap allocated, but reusable
    }

    public async Task ProcessAsync(Memory<byte> data)
    {
        // Memory<T> is allowed in async methods
        await _stream.ReadAsync(data);

        // Convert to Span for synchronous processing
        ProcessSync(data.Span);
    }

    private void ProcessSync(Span<byte> data)
    {
        // Zero-allocation processing
    }
}
```

| Type | Stack-only | Async-safe | Use case |
|------|-----------|------------|----------|
| `Span<T>` | ✅ Yes | ❌ No | Synchronous, high-performance parsing |
| `ReadOnlySpan<T>` | ✅ Yes | ❌ No | Immutable view of data |
| `Memory<T>` | ❌ No | ✅ Yes | Async operations, pooled buffers |
| `ReadOnlyMemory<T>` | ❌ No | ✅ Yes | Immutable async-safe view |

## Why It's a Problem

1. **Hidden allocations**: Simple string operations allocate invisibly.

2. **GC pressure**: High-throughput code spends more time in GC than in logic.

3. **Use-after-free risk**: Manual memory management (pointers) is unsafe.

4. **Performance cliffs**: Code that works at 1k req/sec fails at 100k req/sec.

## Symptoms

- High Gen0/Gen1 GC collection rates in production
- Memory profiler showing string allocations dominating
- Latency spikes correlating with GC pauses
- `Substring`, `Split`, `Trim` calls in hot paths
- Performance degradation under load

## Benchmarks

```csharp
[MemoryDiagnoser]
public class ParsingBenchmarks
{
    private const string Header = "Content-Type: application/json";

    [Benchmark(Baseline = true)]
    public (string, string) ParseWithStrings()
    {
        int colonIndex = Header.IndexOf(':');
        string key = Header.Substring(0, colonIndex);
        string value = Header.Substring(colonIndex + 1).Trim();
        return (key, value);
    }

    [Benchmark]
    public (ReadOnlySpan<char>, ReadOnlySpan<char>) ParseWithSpan()
    {
        ReadOnlySpan<char> header = Header.AsSpan();
        int colonIndex = header.IndexOf(':');
        var key = header[..colonIndex];
        var value = header[(colonIndex + 1)..].Trim();
        return (key, value);
    }
}

// Results:
// |           Method |      Mean | Allocated |
// |----------------- |----------:|----------:|
// | ParseWithStrings | 45.23 ns  |     136 B |
// |    ParseWithSpan |  8.67 ns  |       0 B |
```

## Creating Ref Structs

For custom stack-only types:

```csharp
public ref struct ValueStringBuilder
{
    private Span<char> _buffer;
    private int _position;

    public ValueStringBuilder(Span<char> buffer)
    {
        _buffer = buffer;
        _position = 0;
    }

    public void Append(ReadOnlySpan<char> value)
    {
        value.CopyTo(_buffer[_position..]);
        _position += value.Length;
    }

    public void Append(char c)
    {
        _buffer[_position++] = c;
    }

    public readonly override string ToString()
        => new string(_buffer[.._position]);

    public readonly ReadOnlySpan<char> AsSpan()
        => _buffer[.._position];
}

// Usage
Span<char> buffer = stackalloc char[256];
var builder = new ValueStringBuilder(buffer);
builder.Append("Hello, ");
builder.Append(name);
builder.Append('!');
return builder.ToString();
```

## When to Use Span

| Scenario | Use Span? | Reason |
|----------|-----------|--------|
| Hot path parsing (100k+ ops/sec) | ✅ Yes | Zero allocation matters |
| One-time startup config parsing | ❌ No | Readability > micro-optimization |
| Async stream processing | Use `Memory<T>` | Span can't cross await |
| API return types | ❌ No | Callers expect strings |
| Internal parsing helpers | ✅ Yes | Encapsulate complexity |

## Benefits

- **Zero allocation**: No GC pressure in hot paths
- **Memory safety**: Compiler prevents use-after-free
- **Performance**: 5-10x faster parsing in benchmarks
- **Stack locality**: Better cache performance
- **Type system enforced**: Can't accidentally escape to heap

## See Also

- [Value Semantics](./value-semantics.md)
- [Primitive Obsession](./primitive-obsession.md)
- [Snapshot Immutability (The "ReadOnly" Lie)](./snapshot-immutability.md)
