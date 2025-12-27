# Unsafe Code Patterns

> Access memory directly for maximum performance—use unsafe code judiciously for zero-overhead operations, custom memory layouts, and hardware-level control.

## Problem

Safe C# code has runtime bounds checking, indirect memory access, and managed object overhead. For performance-critical code processing millions of operations per second, these safety checks become bottlenecks. Unsafe code bypasses safety for direct memory manipulation, pointer arithmetic, and unmanaged interop—but requires extreme care to avoid corruption.

## Example

### ❌ Before (Safe, Slower)

```csharp
public class SafeArrayProcessing
{
    public void ProcessArray(byte[] data)
    {
        for (int i = 0; i < data.Length; i++)
        {
            data[i] = (byte)(data[i] * 2);  // Bounds check every iteration
        }
    }
    
    public void CopyData(byte[] source, byte[] dest)
    {
        Array.Copy(source, dest, source.Length);  // Safe but slower
    }
    
    // Problems:
    // - Bounds checking overhead
    // - Indirect memory access
    // - Cannot use SIMD efficiently
}
```

### ✅ After (Unsafe, Faster)

```csharp
public class UnsafeArrayProcessing
{
    public unsafe void ProcessArray(byte[] data)
    {
        fixed (byte* ptr = data)  // Pin array, get pointer
        {
            byte* end = ptr + data.Length;
            
            // No bounds checks, direct memory access
            for (byte* p = ptr; p < end; p++)
            {
                *p = (byte)(*p * 2);
            }
        }
    }
    
    public unsafe void CopyData(byte[] source, byte[] dest)
    {
        fixed (byte* src = source, dst = dest)
        {
            // Fast memory copy
            Buffer.MemoryCopy(src, dst, dest.Length, source.Length);
        }
    }
    
    // ✅ No bounds checks
    // ✅ Direct memory access
    // ✅ Can use SIMD/intrinsics
}
```

## Pointer Basics

```csharp
public unsafe class PointerBasics
{
    // ✅ Pointer declaration and dereferencing
    public void PointerOperations()
    {
        int value = 42;
        int* ptr = &value;       // Get address of value
        int result = *ptr;       // Dereference: read value
        *ptr = 100;              // Dereference: write value
        
        Console.WriteLine(value);  // Prints 100
    }
    
    // ✅ Pointer arithmetic
    public void PointerArithmetic(int* ptr, int length)
    {
        // Increment pointer
        ptr++;           // Moves by sizeof(int) = 4 bytes
        ptr += 10;       // Moves by 10 * sizeof(int) = 40 bytes
        
        // Array-style access
        int value = ptr[5];  // Same as *(ptr + 5)
        
        // Pointer comparison
        int* end = ptr + length;
        while (ptr < end)
        {
            *ptr = 0;
            ptr++;
        }
    }
    
    // ✅ Pointer to different types
    public void TypedPointers()
    {
        int value = 0x12345678;
        int* intPtr = &value;
        byte* bytePtr = (byte*)intPtr;  // Reinterpret as bytes
        
        // Access individual bytes
        byte b0 = bytePtr[0];  // 0x78
        byte b1 = bytePtr[1];  // 0x56
        byte b2 = bytePtr[2];  // 0x34
        byte b3 = bytePtr[3];  // 0x12
    }
}
```

## Fixed Statement

```csharp
public unsafe class FixedPatterns
{
    // ✅ Fixed array
    public void FixedArray(byte[] data)
    {
        fixed (byte* ptr = data)
        {
            // Array is pinned, GC won't move it
            ProcessPointer(ptr, data.Length);
        }
        // Array unpinned here
    }
    
    // ✅ Fixed string
    public void FixedString(string text)
    {
        fixed (char* ptr = text)
        {
            // String is pinned
            for (int i = 0; i < text.Length; i++)
            {
                ptr[i] = char.ToUpper(ptr[i]);  // Modifies string!
            }
        }
    }
    
    // ✅ Multiple fixed pointers
    public void MultipleFixed(byte[] a, byte[] b, byte[] result)
    {
        fixed (byte* pA = a, pB = b, pResult = result)
        {
            for (int i = 0; i < a.Length; i++)
            {
                pResult[i] = (byte)(pA[i] + pB[i]);
            }
        }
    }
    
    // ✅ Fixed buffer in struct
    public struct FixedBuffer
    {
        public fixed byte Data[256];  // Inline fixed-size buffer
    }
    
    public void UseFixedBuffer(FixedBuffer* buffer)
    {
        // Access inline buffer
        for (int i = 0; i < 256; i++)
        {
            buffer->Data[i] = (byte)i;
        }
    }
    
    private void ProcessPointer(byte* ptr, int length) { }
}
```

## Stackalloc

```csharp
public unsafe class StackallocPatterns
{
    // ✅ Stackalloc small buffers
    public void SmallBuffer()
    {
        byte* buffer = stackalloc byte[256];  // On stack, very fast
        
        // Use buffer
        for (int i = 0; i < 256; i++)
        {
            buffer[i] = (byte)i;
        }
        
        // Automatically cleaned up when method returns
    }
    
    // ✅ Stackalloc with Span (safe)
    public void StackallocSpan()
    {
        Span<byte> buffer = stackalloc byte[256];  // Safe wrapper
        
        // Use like array
        for (int i = 0; i < buffer.Length; i++)
        {
            buffer[i] = (byte)i;
        }
    }
    
    // ✅ Conditional stackalloc
    public void ConditionalStackalloc(int size)
    {
        const int MaxStackSize = 512;
        
        Span<byte> buffer = size <= MaxStackSize
            ? stackalloc byte[size]              // Stack
            : new byte[size];                    // Heap
        
        // Use buffer same way
    }
    
    // ❌ Don't stackalloc large/variable buffers
    public void DontDoThis(int size)
    {
        // byte* buffer = stackalloc byte[size];  // Stack overflow risk!
        
        // ✅ Use heap for large/unknown sizes
        byte[] buffer = new byte[size];
    }
}
```

## Struct Layout Control

```csharp
// ✅ Explicit layout for binary protocols
[StructLayout(LayoutKind.Explicit, Pack = 1)]
public struct PacketHeader
{
    [FieldOffset(0)] public byte Version;
    [FieldOffset(1)] public byte MessageType;
    [FieldOffset(2)] public ushort Length;
    [FieldOffset(4)] public uint Sequence;
    [FieldOffset(8)] public long Timestamp;
    
    // Total size: 16 bytes, tightly packed
}

// ✅ Sequential layout with explicit packing
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct TickData
{
    public long Timestamp;
    public decimal Price;
    public int Volume;
    // Pack = 1: no padding between fields
}

// ✅ Union (overlapping fields)
[StructLayout(LayoutKind.Explicit)]
public struct IntFloat
{
    [FieldOffset(0)] public int IntValue;
    [FieldOffset(0)] public float FloatValue;
    
    // Reinterpret bytes as different type
}

public unsafe class StructLayoutUsage
{
    public void ParsePacket(byte[] data)
    {
        fixed (byte* ptr = data)
        {
            // Cast bytes directly to struct (zero-copy)
            PacketHeader* header = (PacketHeader*)ptr;
            
            Console.WriteLine($"Version: {header->Version}");
            Console.WriteLine($"Length: {header->Length}");
        }
    }
}
```

## Memory Manipulation

```csharp
public unsafe class MemoryManipulation
{
    // ✅ Fast memory copy
    public void FastCopy(byte* src, byte* dst, int length)
    {
        Buffer.MemoryCopy(src, dst, length, length);
    }
    
    // ✅ Fast memory set
    public void FastZero(byte* ptr, int length)
    {
        // Zero memory
        for (byte* end = ptr + length; ptr < end; ptr++)
        {
            *ptr = 0;
        }
        
        // Or use platform-specific
        System.Runtime.CompilerServices.Unsafe.InitBlockUnaligned(ptr, 0, (uint)length);
    }
    
    // ✅ Fast memory compare
    public bool FastCompare(byte* a, byte* b, int length)
    {
        byte* endA = a + length;
        
        while (a < endA)
        {
            if (*a != *b)
                return false;
            
            a++;
            b++;
        }
        
        return true;
    }
    
    // ✅ Swap bytes
    public void SwapBytes(byte* a, byte* b)
    {
        byte temp = *a;
        *a = *b;
        *b = temp;
    }
}
```

## Unsafe Span Operations

```csharp
public class UnsafeSpanOps
{
    // ✅ Get pointer from Span
    public unsafe void ProcessSpan(Span<byte> span)
    {
        fixed (byte* ptr = span)
        {
            // Use pointer
            ProcessPointer(ptr, span.Length);
        }
    }
    
    // ✅ Cast Span to different type
    public void ReinterpretSpan()
    {
        Span<byte> bytes = stackalloc byte[16];
        
        // Reinterpret as ints (zero-copy)
        Span<int> ints = MemoryMarshal.Cast<byte, int>(bytes);
        
        ints[0] = 42;  // Modifies underlying bytes
    }
    
    // ✅ Get reference from Span
    public void SpanReference(Span<int> span)
    {
        ref int first = ref MemoryMarshal.GetReference(span);
        first = 100;  // Modifies first element
    }
    
    private unsafe void ProcessPointer(byte* ptr, int length) { }
}
```

## High-Performance Parsing

```csharp
public unsafe class UnsafeParsing
{
    // ✅ Parse FIX message
    public void ParseFIXMessage(byte* data, int length)
    {
        byte* end = data + length;
        byte* current = data;
        
        while (current < end)
        {
            // Find field tag
            int tag = 0;
            while (current < end && *current != '=')
            {
                tag = tag * 10 + (*current - '0');
                current++;
            }
            
            current++;  // Skip '='
            
            // Find field value
            byte* valueStart = current;
            while (current < end && *current != '\x01')  // SOH delimiter
            {
                current++;
            }
            
            int valueLength = (int)(current - valueStart);
            
            // Process field
            ProcessField(tag, valueStart, valueLength);
            
            current++;  // Skip SOH
        }
    }
    
    // ✅ Parse integer from bytes
    public int ParseInt(byte* data, int length)
    {
        int result = 0;
        byte* end = data + length;
        
        while (data < end)
        {
            result = result * 10 + (*data - '0');
            data++;
        }
        
        return result;
    }
    
    // ✅ Parse decimal from bytes
    public decimal ParseDecimal(byte* data, int length)
    {
        long whole = 0;
        long fraction = 0;
        int fractionDigits = 0;
        bool inFraction = false;
        byte* end = data + length;
        
        while (data < end)
        {
            if (*data == '.')
            {
                inFraction = true;
            }
            else if (inFraction)
            {
                fraction = fraction * 10 + (*data - '0');
                fractionDigits++;
            }
            else
            {
                whole = whole * 10 + (*data - '0');
            }
            
            data++;
        }
        
        decimal result = whole;
        
        if (fractionDigits > 0)
        {
            result += fraction / (decimal)Math.Pow(10, fractionDigits);
        }
        
        return result;
    }
    
    private void ProcessField(int tag, byte* value, int length) { }
}
```

## Unmanaged Memory Allocation

```csharp
public unsafe class UnmanagedMemory
{
    // ✅ Allocate unmanaged memory
    public byte* AllocateUnmanaged(int size)
    {
        return (byte*)Marshal.AllocHGlobal(size);
    }
    
    // ✅ Free unmanaged memory
    public void FreeUnmanaged(byte* ptr)
    {
        Marshal.FreeHGlobal((IntPtr)ptr);
    }
    
    // ✅ Managed wrapper for safety
    public class UnmanagedBuffer : IDisposable
    {
        private byte* buffer;
        private readonly int size;
        
        public UnmanagedBuffer(int size)
        {
            this.size = size;
            this.buffer = (byte*)Marshal.AllocHGlobal(size);
        }
        
        public Span<byte> AsSpan() => new Span<byte>(buffer, size);
        
        public void Dispose()
        {
            if (buffer != null)
            {
                Marshal.FreeHGlobal((IntPtr)buffer);
                buffer = null;
            }
        }
    }
}
```

## Best Practices

### 1. Validate Bounds Manually

```csharp
// ✅ Manual bounds checking
public unsafe void SafeUnsafe(byte* ptr, int length, int index)
{
    if (index < 0 || index >= length)
        throw new IndexOutOfRangeException();
    
    ptr[index] = 42;
}
```

### 2. Use Fixed for Managed Arrays

```csharp
// ✅ Pin before using pointer
fixed (byte* ptr = managedArray)
{
    // Safe: array won't move
}

// ❌ Don't use pointer after fixed block
```

### 3. Prefer Span When Possible

```csharp
// ✅ Span provides safety
public void SafeAlternative(Span<byte> data)
{
    // Bounds checked, but fast
}

// Only use unsafe when Span insufficient
```

### 4. Document Safety Assumptions

```csharp
/// <summary>
/// SAFETY: Assumes ptr points to valid memory of at least length bytes.
/// Caller must ensure ptr remains valid for duration of call.
/// </summary>
public unsafe void ProcessPointer(byte* ptr, int length) { }
```

## Performance Characteristics

- **Bounds check elimination**: 2-5% speedup
- **Direct memory access**: 10-20% speedup
- **Pointer arithmetic**: Fastest possible
- **Stack allocation**: 100x faster than heap

## When to Use

- Performance-critical inner loops
- Binary protocol parsing
- Interop with unmanaged code
- Custom memory layouts
- Zero-allocation scenarios

## When NOT to Use

- General application code
- When Span<T> suffices
- Maintenance-critical code
- Learning/prototyping

## Related Patterns

- [Memory Reinterpretation](./memory-reinterpretation.md) — Type punning
- [Zero-Copy Techniques](./zero-copy-techniques.md) — Span usage
- [SIMD Vectorization](./simd-vectorization.md) — Vector operations
- [Intrinsics](./intrinsics-hardware-acceleration.md) — Hardware acceleration

## References

- "Unsafe Code and Pointers" - C# Programming Guide
- "High Performance C#" - Pro .NET Memory Management
- "Span<T> and Memory<T>" - Microsoft Docs
