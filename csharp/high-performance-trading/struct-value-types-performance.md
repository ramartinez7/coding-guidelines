# Struct and Value Types for Performance

> Use value types strategically to reduce heap allocations and improve locality—understanding when structs help vs when they hurt performance.

## Problem

Reference types (classes) allocate on the heap, require garbage collection, and add pointer indirection. Value types (structs) allocate on the stack or inline in containing objects, avoiding GC pressure. However, large structs or incorrect usage can hurt performance through excessive copying. The key is knowing when to use each.

## Example

### ❌ Before (Class for Small Data)

```csharp
// ❌ Class allocates on heap
public class Point
{
    public double X { get; set; }
    public double Y { get; set; }
}

public class ShapeRenderer
{
    public void DrawLine(Point start, Point end)
    {
        // 2 heap allocations per call
        // Pointer indirection on access
        // GC pressure accumulates
    }

    public void DrawShape(Point[] points)
    {
        // Array of pointers to heap objects
        // Poor cache locality
        // Many cache misses
    }
}

// At 100k calls/sec: 200k allocations/sec
```

### ✅ After (Struct for Small Data)

```csharp
// ✅ Struct allocates on stack or inline
public readonly struct Point
{
    public double X { get; }
    public double Y { get; }

    public Point(double x, double y)
    {
        this.X = x;
        this.Y = y;
    }
}

public class ShapeRenderer
{
    public void DrawLine(Point start, Point end)
    {
        // Zero heap allocations
        // Values passed directly
        // No GC pressure
    }

    public void DrawShape(Point[] points)
    {
        // Array of values (not pointers)
        // Excellent cache locality
        // Sequential memory access
    }
}

// At 100k calls/sec: 0 allocations
```

## When to Use Structs

```csharp
// ✅ Small, immutable data (≤16 bytes recommended)
public readonly struct Money
{
    public decimal Amount { get; }
    public string Currency { get; }  // 8-byte pointer to string

    // 16 bytes total (decimal) + 8 bytes (pointer) = 24 bytes
    // Still reasonable for a struct
}

// ✅ Frequently created short-lived objects
public readonly struct TickData
{
    public long Timestamp { get; }
    public decimal Price { get; }
    public int Volume { get; }

    // Created millions of times, never stored long-term
    // Perfect for struct
}

// ✅ Value semantics (equality by value)
public readonly struct OrderId
{
    public Guid Value { get; }

    // Identity is the value itself
    // Perfect for struct
}
```

## When NOT to Use Structs

```csharp
// ❌ Large structs (>16 bytes) passed frequently
public struct LargeData  // ❌ 80 bytes
{
    public decimal Field1;  // 16 bytes
    public decimal Field2;  // 16 bytes
    public decimal Field3;  // 16 bytes
    public decimal Field4;  // 16 bytes
    public decimal Field5;  // 16 bytes
}

public void Process(LargeData data)  // Copies 80 bytes!
{
    // Every call copies 80 bytes to stack
    // Worse than passing reference (8 bytes)
}

// ✅ Use class or pass by ref
public class LargeData { /* ... */ }
public void Process(LargeData data) { }  // 8-byte pointer

// ✅ Or pass struct by ref
public void Process(ref LargeData data) { }  // 8-byte reference

// ❌ Mutable structs (causes confusion)
public struct MutablePoint
{
    public double X { get; set; }
    public double Y { get; set; }
}

var points = new MutablePoint[100];
points[0].X = 10;  // OK

MutablePoint temp = points[0];
temp.X = 20;  // Modifies copy, not array element!

// ✅ Use readonly struct
public readonly struct ImmutablePoint
{
    public double X { get; }
    public double Y { get; }

    public Point WithX(double x) => new Point(x, Y);
}

// ❌ Structs with reference type fields (defeats purpose)
public struct BadStruct
{
    public string Name;  // Reference to heap
    public List<int> Items;  // Reference to heap

    // Still causes heap allocations and GC pressure
}
```

## Readonly Struct Pattern

```csharp
// ✅ Readonly struct prevents defensive copies
public readonly struct Price
{
    public decimal Value { get; }
    public string Currency { get; }

    public Price(decimal value, string currency)
    {
        this.Value = value;
        this.Currency = currency;
    }

    // Operators work efficiently
    public static Price operator +(Price a, Price b)
    {
        if (a.Currency != b.Currency)
            throw new InvalidOperationException("Currency mismatch");

        return new Price(a.Value + b.Value, a.Currency);
    }
}

// Compiler knows Price can't mutate, avoids defensive copies
public decimal CalculateTotal(Price[] prices)
{
    decimal sum = 0;
    foreach (var price in prices)  // No defensive copy!
    {
        sum += price.Value;
    }
    return sum;
}
```

## Ref Struct for Stack-Only Types

```csharp
// ✅ Ref struct: can't escape to heap
public ref struct StackBuffer
{
    private Span<byte> buffer;

    public StackBuffer(Span<byte> buffer)
    {
        this.buffer = buffer;
    }

    public void Write(byte value)
    {
        buffer[0] = value;
    }
}

// Cannot:
// - Store in fields of regular class
// - Use in async methods
// - Box to object
// - Store in List<T>

// Perfect for stack-allocated buffers
Span<byte> stack = stackalloc byte[256];
var buffer = new StackBuffer(stack);
buffer.Write(42);
```

## Inline Arrays (C# 12)

```csharp
// ✅ Fixed-size array as value type
[InlineArray(16)]
public struct PriceBuffer
{
    private decimal element0;

    // Compiler generates efficient inline array
    // 16 * 16 = 256 bytes allocated inline
}

public class OrderBook
{
    private PriceBuffer prices;  // 256 bytes inline, no heap allocation

    public void UpdatePrice(int index, decimal price)
    {
        prices[index] = price;  // Direct access, no bounds check
    }
}
```

## Boxing Avoidance

```csharp
// ❌ Boxing allocates on heap
public struct Point
{
    public int X, Y;
}

object boxed = new Point { X = 10, Y = 20 };  // Boxing allocation!

// ✅ Avoid boxing with generics
public void Process<T>(T value) where T : struct
{
    // No boxing!
}

// ✅ Or use interfaces on structs carefully
public interface IPosition
{
    int X { get; }
    int Y { get; }
}

public readonly struct Point : IPosition
{
    public int X { get; }
    public int Y { get; }
}

public void Process(IPosition pos)  // ❌ Boxing!
{
    Console.WriteLine(pos.X);
}

public void Process<T>(T pos) where T : struct, IPosition  // ✅ No boxing
{
    Console.WriteLine(pos.X);
}
```

## Struct Layout for Performance

```csharp
// ✅ Explicit layout for optimal packing
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct MarketData
{
    public long Timestamp;   // 8 bytes
    public decimal Price;    // 16 bytes
    public int Volume;       // 4 bytes
    public byte Flags;       // 1 byte
    // Total: 29 bytes (no padding with Pack = 1)
}

// ✅ Or optimize for alignment
[StructLayout(LayoutKind.Explicit)]
public struct AlignedMarketData
{
    [FieldOffset(0)]
    public long Timestamp;   // 8-byte aligned

    [FieldOffset(8)]
    public decimal Price;    // 16 bytes

    [FieldOffset(24)]
    public int Volume;       // 4-byte aligned

    [FieldOffset(28)]
    public byte Flags;

    // 32 bytes total (cache-friendly)
}
```

## Passing Structs Efficiently

```csharp
public readonly struct LargeStruct
{
    // 40 bytes total
    public decimal Value1 { get; }
    public decimal Value2 { get; }
    public long Timestamp { get; }
}

// ❌ Pass by value: copies 40 bytes
public void Process(LargeStruct data)
{
    // ...
}

// ✅ Pass by readonly ref: copies 8 bytes (pointer)
public void Process(in LargeStruct data)
{
    // 'in' prevents modification and avoids copy
    // data is readonly reference
}

// ✅ Return by readonly ref
private LargeStruct[] buffer = new LargeStruct[100];

public ref readonly LargeStruct GetAt(int index)
{
    return ref buffer[index];  // No copy!
}

// Usage
ref readonly var data = ref GetAt(50);
Process(in data);  // Efficient!
```

## Struct Equality

```csharp
// ❌ Default struct equality is slow (reflection)
public struct Point
{
    public int X, Y;
}

var p1 = new Point { X = 10, Y = 20 };
var p2 = new Point { X = 10, Y = 20 };
bool equal = p1.Equals(p2);  // Uses reflection!

// ✅ Implement IEquatable<T>
public readonly struct Point : IEquatable<Point>
{
    public int X { get; }
    public int Y { get; }

    public bool Equals(Point other)
    {
        return X == other.X && Y == other.Y;
    }

    public override bool Equals(object? obj)
    {
        return obj is Point point && Equals(point);
    }

    public override int GetHashCode()
    {
        return HashCode.Combine(X, Y);
    }

    public static bool operator ==(Point left, Point right)
    {
        return left.Equals(right);
    }

    public static bool operator !=(Point left, Point right)
    {
        return !left.Equals(right);
    }
}
```

## Best Practices

### 1. Keep Structs Small (≤16 bytes ideal)

```csharp
// ✅ Small struct
public readonly struct OrderId
{
    public long Value { get; }
    // 8 bytes
}

// ⚠️ Medium struct - consider ref passing
public readonly struct Quote
{
    public decimal Bid { get; }
    public decimal Ask { get; }
    // 32 bytes - use 'in' when passing
}

// ❌ Large struct - use class instead
public readonly struct LargeOrder
{
    // 200+ bytes
    // Should be a class!
}
```

### 2. Make Structs Readonly

```csharp
// ✅ Readonly struct
public readonly struct Price { /* ... */ }

// ❌ Mutable struct
public struct MutablePrice { /* ... */ }
```

### 3. Use In Parameters for Large Structs

```csharp
public void Process(in LargeStruct data)
{
    // Passed by reference, readonly
}
```

### 4. Avoid Structs with Reference Fields

```csharp
// ❌ Defeats purpose of struct
public struct BadStruct
{
    public string Name;  // Heap allocation
    public List<int> Items;  // Heap allocation
}

// ✅ All value fields
public struct GoodStruct
{
    public int Id;
    public decimal Value;
    public long Timestamp;
}
```

## Performance Characteristics

- **Allocation**: Stack (zero GC pressure)
- **Cache locality**: Excellent (inline in arrays)
- **Copy cost**: O(size) - problematic for large structs
- **Equality**: Fast with IEquatable<T>, slow without

## When to Use

- Small immutable data (≤16-32 bytes)
- High-frequency short-lived objects
- Value semantics required
- Arrays of data (better cache locality)

## When NOT to Use

- Large data structures (>32 bytes)
- Mutable state
- Polymorphic behavior needed
- Long-lived objects

## Related Patterns

- [Value Semantics](../patterns/value-semantics.md) — Value vs reference types
- [Struct Layout](../patterns/struct-layout.md) — Memory layout optimization
- [Memory Safety (Span)](../patterns/memory-safety-span.md) — Stack allocation
- [Allocation Budget](../patterns/allocation-budget.md) — Zero allocations

## References

- "Performance Traps of ref locals and ref returns in C#" - Microsoft Docs
- "Writing High-Performance .NET Code" by Ben Watson
- "Pro .NET Memory Management" by Konrad Kokosa
