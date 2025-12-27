# Zero-Copy Techniques

> Avoid copying data between buffers using Span<T>, Memory<T>, and ref returns—minimize memory bandwidth and improve cache efficiency.

## Problem

Traditional code copies data multiple times: from network buffer to parse buffer, parse buffer to domain objects, domain objects to response buffer. Each copy wastes CPU cycles and memory bandwidth. In high-frequency systems, these copies accumulate into significant overhead—both in latency and throughput.

## Example

### ❌ Before (Multiple Copies)

```csharp
public class MessageProcessor
{
    public OrderMessage ProcessMessage(byte[] networkBuffer)
    {
        // Copy 1: Extract message portion
        var messageBytes = new byte[networkBuffer.Length - 4];
        Array.Copy(networkBuffer, 4, messageBytes, 0, messageBytes.Length);

        // Copy 2: Parse into string
        var messageString = Encoding.UTF8.GetString(messageBytes);

        // Copy 3: Parse fields
        var parts = messageString.Split('|');
        var orderId = parts[0];
        var symbol = parts[1];
        var price = decimal.Parse(parts[2]);

        // Copy 4: Create object
        return new OrderMessage
        {
            OrderId = orderId,
            Symbol = symbol,
            Price = price
        };
    }
}

// 4 copies for a single message!
// Each copy allocates memory and stresses GC
```

### ✅ After (Zero-Copy with Span)

```csharp
public class MessageProcessor
{
    public OrderMessage ProcessMessage(ReadOnlySpan<byte> networkBuffer)
    {
        // Zero-copy: work directly on buffer
        var messageSpan = networkBuffer[4..];

        // Zero-copy: decode in place
        Span<char> charBuffer = stackalloc char[messageSpan.Length];
        var charCount = Encoding.UTF8.GetChars(messageSpan, charBuffer);
        var messageChars = charBuffer[..charCount];

        // Zero-copy: split using spans
        var firstPipe = messageChars.IndexOf('|');
        var secondPipe = messageChars[(firstPipe + 1)..].IndexOf('|') + firstPipe + 1;

        var orderIdSpan = messageChars[..firstPipe];
        var symbolSpan = messageChars[(firstPipe + 1)..secondPipe];
        var priceSpan = messageChars[(secondPipe + 1)..];

        // Only allocate final result
        return new OrderMessage
        {
            OrderId = new string(orderIdSpan),
            Symbol = new string(symbolSpan),
            Price = decimal.Parse(priceSpan)
        };
    }
}

// Single allocation at the end
// All intermediate work on stack
```

## Span<T> for Buffer Slicing

```csharp
public readonly ref struct FIXMessage
{
    private readonly ReadOnlySpan<byte> buffer;

    public FIXMessage(ReadOnlySpan<byte> buffer)
    {
        this.buffer = buffer;
    }

    // ✅ Zero-copy access to fields
    public ReadOnlySpan<byte> GetField(int tag)
    {
        var tagPrefix = $"{tag}=";
        var index = buffer.IndexOf(Encoding.UTF8.GetBytes(tagPrefix));

        if (index < 0)
            return ReadOnlySpan<byte>.Empty;

        var start = index + tagPrefix.Length;
        var end = buffer[start..].IndexOf((byte)'\x01');

        if (end < 0)
            return buffer[start..];

        return buffer.Slice(start, end);
    }

    public string GetStringField(int tag)
    {
        var fieldBytes = GetField(tag);
        return Encoding.UTF8.GetString(fieldBytes);
    }

    public int GetIntField(int tag)
    {
        var fieldBytes = GetField(tag);
        Span<char> chars = stackalloc char[fieldBytes.Length];
        Encoding.UTF8.GetChars(fieldBytes, chars);
        return int.Parse(chars);
    }
}

// Usage: zero copies until final conversion
var message = new FIXMessage(networkBuffer);
var symbol = message.GetStringField(55);  // Only symbol is allocated
var price = message.GetIntField(44);  // Parsed from span
```

## Memory<T> for Async Operations

```csharp
// ❌ Can't use Span<T> in async methods
public async Task<int> ProcessAsync(Span<byte> buffer)  // Error: can't use Span in async
{
    await Task.Delay(100);
    return buffer.Length;
}

// ✅ Use Memory<T> for async scenarios
public async Task<int> ProcessAsync(Memory<byte> buffer)
{
    // Memory<T> can be used across await boundaries
    await Task.Delay(100);

    // Convert to Span when needed
    var span = buffer.Span;
    return ProcessSpan(span);
}

private int ProcessSpan(Span<byte> buffer)
{
    // Do work with span
    return buffer.Length;
}
```

## Ref Returns for Direct Access

```csharp
public class PriceBook
{
    private decimal[] prices = new decimal[1000];

    // ❌ Returns copy
    public decimal GetPrice(int index)
    {
        return prices[index];
    }

    // ✅ Returns reference (zero-copy)
    public ref decimal GetPriceRef(int index)
    {
        return ref prices[index];
    }

    // ✅ Read-only reference
    public ref readonly decimal GetPriceReadonly(int index)
    {
        return ref prices[index];
    }
}

// Usage
var book = new PriceBook();

// ❌ Copy
decimal price = book.GetPrice(100);
price = price * 1.1m;  // Modifies local copy

// ✅ Direct access
ref decimal priceRef = ref book.GetPriceRef(100);
priceRef = priceRef * 1.1m;  // Modifies array directly!

// ✅ Read-only (safe)
ref readonly decimal priceRO = ref book.GetPriceReadonly(100);
var calculated = priceRO * 1.1m;  // Can read, but can't modify
```

## MemoryMarshal for Unsafe Casts

```csharp
public static class ZeroCopyParsing
{
    // ✅ Cast byte span to struct span (zero-copy)
    public static ReadOnlySpan<Quote> ParseQuotes(ReadOnlySpan<byte> buffer)
    {
        // Cast bytes directly to Quote structs (unsafe but fast)
        return MemoryMarshal.Cast<byte, Quote>(buffer);
    }

    // ✅ Get reference to first element
    public static ref Quote GetFirstQuote(Span<Quote> quotes)
    {
        return ref MemoryMarshal.GetReference(quotes);
    }

    // ✅ Write struct to bytes without allocating
    public static void WriteQuote(Quote quote, Span<byte> destination)
    {
        var quoteSpan = MemoryMarshal.CreateReadOnlySpan(ref quote, 1);
        var bytes = MemoryMarshal.AsBytes(quoteSpan);
        bytes.CopyTo(destination);
    }
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct Quote
{
    public long Timestamp;
    public decimal BidPrice;
    public decimal AskPrice;
    public int BidSize;
    public int AskSize;
}
```

## String Parsing Without Allocation

```csharp
public static class ZeroCopyParsing
{
    // ✅ Parse int from span
    public static bool TryParseInt32(ReadOnlySpan<char> span, out int value)
    {
        return int.TryParse(span, out value);
    }

    // ✅ Parse decimal from span
    public static bool TryParseDecimal(ReadOnlySpan<char> span, out decimal value)
    {
        return decimal.TryParse(span, out value);
    }

    // ✅ Split without allocating
    public static void Split(
        ReadOnlySpan<char> input,
        char separator,
        Span<Range> ranges,
        out int rangeCount)
    {
        rangeCount = 0;
        int start = 0;

        for (int i = 0; i < input.Length && rangeCount < ranges.Length; i++)
        {
            if (input[i] == separator)
            {
                ranges[rangeCount++] = new Range(start, i);
                start = i + 1;
            }
        }

        if (start < input.Length && rangeCount < ranges.Length)
        {
            ranges[rangeCount++] = new Range(start, input.Length);
        }
    }
}

// Usage
ReadOnlySpan<char> csv = "100|MSFT|340.50|1000";
Span<Range> ranges = stackalloc Range[4];
ZeroCopyParsing.Split(csv, '|', ranges, out var count);

var orderIdSpan = csv[ranges[0]];
var symbolSpan = csv[ranges[1]];
var priceSpan = csv[ranges[2]];
var quantitySpan = csv[ranges[3]];

int.TryParse(orderIdSpan, out var orderId);
decimal.TryParse(priceSpan, out var price);
int.TryParse(quantitySpan, out var quantity);
```

## ArraySegment for Sub-Arrays

```csharp
// ❌ Copies array
public void ProcessSubArray(byte[] data, int offset, int count)
{
    var subArray = new byte[count];
    Array.Copy(data, offset, subArray, 0, count);
    Process(subArray);
}

// ✅ ArraySegment: zero-copy view
public void ProcessSubArray(ArraySegment<byte> segment)
{
    // No copy, just offset + count
    Process(segment);
}

// ✅ Even better: use Span
public void ProcessSubArray(ReadOnlySpan<byte> data)
{
    Process(data);
}

// Usage
byte[] buffer = GetBuffer();
ProcessSubArray(buffer.AsSpan(10, 100));  // Zero-copy slice
```

## Pooled Buffers with Span

```csharp
public class ZeroCopyProcessor
{
    private readonly ArrayPool<byte> pool = ArrayPool<byte>.Shared;

    public void Process(ReadOnlySpan<byte> input)
    {
        // Rent buffer from pool
        var buffer = pool.Rent(input.Length * 2);

        try
        {
            // Work with buffer as span (zero-copy)
            var bufferSpan = buffer.AsSpan(0, input.Length * 2);
            input.CopyTo(bufferSpan);

            // Transform in place
            for (int i = 0; i < input.Length; i++)
            {
                bufferSpan[i] = (byte)(bufferSpan[i] ^ 0xFF);
            }

            // Use transformed data
            ProcessTransformed(bufferSpan[..input.Length]);
        }
        finally
        {
            pool.Return(buffer);
        }
    }
}
```

## Best Practices

### 1. Use Span<T> for Sync, Memory<T> for Async

```csharp
// ✅ Synchronous: use Span
public int ParseMessage(ReadOnlySpan<byte> buffer)
{
    return ProcessBuffer(buffer);
}

// ✅ Asynchronous: use Memory
public async Task<int> ParseMessageAsync(ReadOnlyMemory<byte> buffer)
{
    await Task.Delay(100);
    return ProcessBuffer(buffer.Span);
}
```

### 2. Stack Allocate Small Buffers

```csharp
// ✅ Small fixed-size buffers on stack
Span<byte> buffer = stackalloc byte[256];

// ❌ Don't stack allocate large or variable buffers
// Span<byte> buffer = stackalloc byte[size];  // Can overflow stack!
```

### 3. Use ReadOnlySpan for Immutability

```csharp
// ✅ Prevent accidental modification
public void ProcessData(ReadOnlySpan<byte> data)
{
    // data[0] = 42;  // Compile error!
}
```

### 4. Avoid String Allocation

```csharp
// ❌ Allocates string
var substring = myString.Substring(0, 10);

// ✅ Zero-copy slice
var substring = myString.AsSpan(0, 10);
```

## Performance Characteristics

- **Memory copies**: Eliminated
- **Allocations**: Reduced to minimum necessary
- **Cache efficiency**: Improved (fewer memory accesses)
- **Throughput**: 2-10x improvement in parsing workloads

## When to Use

- High-frequency parsing or serialization
- Large buffers or strings
- Pipeline processing with intermediate buffers
- Network protocol parsing

## When NOT to Use

- Need to store references across async calls (use Memory<T>)
- Data lifetime extends beyond current scope
- Simple code readability more important than performance

## Related Patterns

- [Memory Safety (Span)](../patterns/memory-safety-span.md) — Detailed Span usage
- [Memory Pooling Strategies](./memory-pooling-strategies.md) — Pool buffers
- [Hot Path Optimization](./hot-path-optimization.md) — Optimize critical paths
- [Allocation Budget](../patterns/allocation-budget.md) — Zero allocations

## References

- "Span<T>" - Microsoft Docs
- "Memory<T> and Span<T> usage guidelines" - .NET Blog
- "Span<T> - Adam Sitnik" - Performance series
