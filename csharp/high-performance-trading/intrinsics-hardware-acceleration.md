# CPU Intrinsics and Hardware Acceleration

> Directly leverage CPU-specific instructions for maximum performance—access SSE, AVX, AVX2, AVX-512, BMI, and AES instructions unavailable in standard C#.

## Problem

Standard C# code compiles to generic instructions that work on all CPUs. This leaves powerful CPU-specific instructions unused—specialized operations for cryptography, bit manipulation, vector math, and string processing. High-performance code needs direct access to these hardware features for maximum throughput and minimum latency.

## Example

### ❌ Before (Standard C#)

```csharp
public class StandardProcessing
{
    // ❌ Generic code, misses hardware acceleration
    public void AddArrays(float[] a, float[] b, float[] result)
    {
        for (int i = 0; i < a.Length; i++)
        {
            result[i] = a[i] + b[i];
        }
        
        // Scalar operations: 1 float per instruction
        // AVX2 could do 8 floats per instruction!
    }
    
    // ❌ Missing fast bit manipulation
    public int CountBits(uint value)
    {
        int count = 0;
        while (value != 0)
        {
            count += (int)(value & 1);
            value >>= 1;
        }
        return count;
        
        // ~32 iterations, POPCNT does it in 1 instruction!
    }
}
```

### ✅ After (Hardware Intrinsics)

```csharp
using System.Runtime.Intrinsics;
using System.Runtime.Intrinsics.X86;

public class IntrinsicsProcessing
{
    // ✅ AVX2: 8 floats per instruction
    public void AddArrays(float[] a, float[] b, float[] result)
    {
        if (!Avx.IsSupported)
        {
            AddArraysScalar(a, b, result);
            return;
        }
        
        int i = 0;
        
        unsafe
        {
            fixed (float* pa = a, pb = b, pr = result)
            {
                // Process 8 floats at once
                for (; i <= a.Length - 8; i += 8)
                {
                    Vector256<float> va = Avx.LoadVector256(pa + i);
                    Vector256<float> vb = Avx.LoadVector256(pb + i);
                    Vector256<float> vr = Avx.Add(va, vb);
                    Avx.Store(pr + i, vr);
                }
            }
        }
        
        // Handle remainder
        for (; i < a.Length; i++)
        {
            result[i] = a[i] + b[i];
        }
    }
    
    // ✅ POPCNT: 1 instruction
    public int CountBits(uint value)
    {
        if (Popcnt.IsSupported)
        {
            return (int)Popcnt.PopCount(value);
        }
        
        return CountBitsScalar(value);
    }
    
    private void AddArraysScalar(float[] a, float[] b, float[] result)
    {
        for (int i = 0; i < a.Length; i++)
            result[i] = a[i] + b[i];
    }
    
    private int CountBitsScalar(uint value)
    {
        int count = 0;
        while (value != 0)
        {
            count += (int)(value & 1);
            value >>= 1;
        }
        return count;
    }
}
```

## SSE/AVX Vector Operations

```csharp
public static class VectorIntrinsics
{
    // ✅ SSE: 4 floats at once
    public static unsafe void AddSSE(float* a, float* b, float* result, int length)
    {
        if (!Sse.IsSupported)
            throw new NotSupportedException();
        
        int i = 0;
        for (; i <= length - 4; i += 4)
        {
            Vector128<float> va = Sse.LoadVector128(a + i);
            Vector128<float> vb = Sse.LoadVector128(b + i);
            Vector128<float> vr = Sse.Add(va, vb);
            Sse.Store(result + i, vr);
        }
    }
    
    // ✅ AVX2: 8 floats at once
    public static unsafe void AddAVX2(float* a, float* b, float* result, int length)
    {
        if (!Avx.IsSupported)
            throw new NotSupportedException();
        
        int i = 0;
        for (; i <= length - 8; i += 8)
        {
            Vector256<float> va = Avx.LoadVector256(a + i);
            Vector256<float> vb = Avx.LoadVector256(b + i);
            Vector256<float> vr = Avx.Add(va, vb);
            Avx.Store(result + i, vr);
        }
    }
    
    // ✅ Fused Multiply-Add (FMA)
    public static unsafe void FusedMultiplyAdd(
        float* a, float* b, float* c, float* result, int length)
    {
        if (!Fma.IsSupported)
            throw new NotSupportedException();
        
        int i = 0;
        for (; i <= length - 8; i += 8)
        {
            Vector256<float> va = Avx.LoadVector256(a + i);
            Vector256<float> vb = Avx.LoadVector256(b + i);
            Vector256<float> vc = Avx.LoadVector256(c + i);
            
            // result = (a * b) + c in one instruction!
            Vector256<float> vr = Fma.MultiplyAdd(va, vb, vc);
            Avx.Store(result + i, vr);
        }
    }
    
    // ✅ Horizontal sum (sum all lanes)
    public static unsafe float HorizontalSum(float* values, int length)
    {
        if (!Avx.IsSupported)
            throw new NotSupportedException();
        
        Vector256<float> sum = Vector256<float>.Zero;
        int i = 0;
        
        for (; i <= length - 8; i += 8)
        {
            Vector256<float> v = Avx.LoadVector256(values + i);
            sum = Avx.Add(sum, v);
        }
        
        // Extract and sum all 8 lanes
        float result = 0;
        for (int j = 0; j < 8; j++)
        {
            result += sum.GetElement(j);
        }
        
        // Handle remainder
        for (; i < length; i++)
        {
            result += values[i];
        }
        
        return result;
    }
}
```

## Bit Manipulation Intrinsics

```csharp
public static class BitIntrinsics
{
    // ✅ Population count (count set bits)
    public static int PopCount(ulong value)
    {
        if (Popcnt.X64.IsSupported)
        {
            return (int)Popcnt.X64.PopCount(value);
        }
        
        return PopCountScalar(value);
    }
    
    // ✅ Trailing zero count (find first set bit)
    public static int TrailingZeroCount(ulong value)
    {
        if (Bmi1.X64.IsSupported)
        {
            return (int)Bmi1.X64.TrailingZeroCount(value);
        }
        
        return TrailingZeroCountScalar(value);
    }
    
    // ✅ Leading zero count
    public static int LeadingZeroCount(ulong value)
    {
        if (Lzcnt.X64.IsSupported)
        {
            return (int)Lzcnt.X64.LeadingZeroCount(value);
        }
        
        return LeadingZeroCountScalar(value);
    }
    
    // ✅ Extract bit field
    public static ulong ExtractBits(ulong value, byte start, byte length)
    {
        if (Bmi1.X64.IsSupported)
        {
            return Bmi1.X64.BitFieldExtract(value, start, length);
        }
        
        ulong mask = (1ul << length) - 1;
        return (value >> start) & mask;
    }
    
    // ✅ Parallel bit deposit (scatter bits)
    public static ulong ParallelBitDeposit(ulong value, ulong mask)
    {
        if (Bmi2.X64.IsSupported)
        {
            return Bmi2.X64.ParallelBitDeposit(value, mask);
        }
        
        return ParallelBitDepositScalar(value, mask);
    }
    
    // ✅ Parallel bit extract (gather bits)
    public static ulong ParallelBitExtract(ulong value, ulong mask)
    {
        if (Bmi2.X64.IsSupported)
        {
            return Bmi2.X64.ParallelBitExtract(value, mask);
        }
        
        return ParallelBitExtractScalar(value, mask);
    }
    
    private static int PopCountScalar(ulong value)
    {
        int count = 0;
        while (value != 0)
        {
            count++;
            value &= value - 1;
        }
        return count;
    }
    
    private static int TrailingZeroCountScalar(ulong value)
    {
        if (value == 0) return 64;
        int count = 0;
        while ((value & 1) == 0)
        {
            count++;
            value >>= 1;
        }
        return count;
    }
    
    private static int LeadingZeroCountScalar(ulong value)
    {
        if (value == 0) return 64;
        int count = 0;
        while ((value & (1ul << 63)) == 0)
        {
            count++;
            value <<= 1;
        }
        return count;
    }
    
    private static ulong ParallelBitDepositScalar(ulong value, ulong mask)
    {
        ulong result = 0;
        int k = 0;
        
        for (int i = 0; i < 64; i++)
        {
            if ((mask & (1ul << i)) != 0)
            {
                if ((value & (1ul << k)) != 0)
                {
                    result |= 1ul << i;
                }
                k++;
            }
        }
        
        return result;
    }
    
    private static ulong ParallelBitExtractScalar(ulong value, ulong mask)
    {
        ulong result = 0;
        int k = 0;
        
        for (int i = 0; i < 64; i++)
        {
            if ((mask & (1ul << i)) != 0)
            {
                if ((value & (1ul << i)) != 0)
                {
                    result |= 1ul << k;
                }
                k++;
            }
        }
        
        return result;
    }
}
```

## AES Encryption Intrinsics

```csharp
public static class AESIntrinsics
{
    // ✅ Hardware-accelerated AES encryption
    public static unsafe Vector128<byte> EncryptBlock(
        Vector128<byte> block, Vector128<byte> key)
    {
        if (!Aes.IsSupported)
            throw new NotSupportedException("AES instructions not available");
        
        // Single AES round (much faster than software)
        return Aes.Encrypt(block, key);
    }
    
    // ✅ Last round
    public static Vector128<byte> EncryptLastRound(
        Vector128<byte> block, Vector128<byte> key)
    {
        if (!Aes.IsSupported)
            throw new NotSupportedException();
        
        return Aes.EncryptLast(block, key);
    }
    
    // ✅ Decrypt
    public static Vector128<byte> DecryptBlock(
        Vector128<byte> block, Vector128<byte> key)
    {
        if (!Aes.IsSupported)
            throw new NotSupportedException();
        
        return Aes.Decrypt(block, key);
    }
}
```

## CRC32 Intrinsics

```csharp
public static class CRC32Intrinsics
{
    // ✅ Hardware-accelerated CRC32
    public static uint CalculateCRC32(ReadOnlySpan<byte> data)
    {
        if (!Sse42.IsSupported)
        {
            return CalculateCRC32Software(data);
        }
        
        uint crc = 0xFFFFFFFF;
        
        unsafe
        {
            fixed (byte* ptr = data)
            {
                byte* current = ptr;
                int remaining = data.Length;
                
                // Process 8 bytes at a time on x64
                if (Sse42.X64.IsSupported)
                {
                    while (remaining >= 8)
                    {
                        ulong value = *(ulong*)current;
                        crc = (uint)Sse42.X64.Crc32(crc, value);
                        current += 8;
                        remaining -= 8;
                    }
                }
                
                // Process 4 bytes at a time
                while (remaining >= 4)
                {
                    uint value = *(uint*)current;
                    crc = Sse42.Crc32(crc, value);
                    current += 4;
                    remaining -= 4;
                }
                
                // Process remaining bytes
                while (remaining > 0)
                {
                    crc = Sse42.Crc32(crc, *current);
                    current++;
                    remaining--;
                }
            }
        }
        
        return ~crc;
    }
    
    private static uint CalculateCRC32Software(ReadOnlySpan<byte> data)
    {
        // Software fallback implementation
        uint crc = 0xFFFFFFFF;
        foreach (byte b in data)
        {
            crc ^= b;
            for (int i = 0; i < 8; i++)
            {
                crc = (crc >> 1) ^ (0xEDB88320 & ~((crc & 1) - 1));
            }
        }
        return ~crc;
    }
}
```

## Price Processing with Intrinsics

```csharp
public class PriceProcessor
{
    // ✅ Calculate VWAP using AVX
    public unsafe decimal CalculateVWAP(float* prices, int* volumes, int count)
    {
        if (!Avx.IsSupported)
        {
            return CalculateVWAPScalar(prices, volumes, count);
        }
        
        Vector256<float> sumPriceVolume = Vector256<float>.Zero;
        Vector256<float> sumVolume = Vector256<float>.Zero;
        
        int i = 0;
        for (; i <= count - 8; i += 8)
        {
            Vector256<float> vPrices = Avx.LoadVector256(prices + i);
            
            // Convert int volumes to float (using AVX2)
            Vector256<int> vVolumesInt = Avx.LoadVector256(volumes + i);
            Vector256<float> vVolumes = Avx.ConvertToVector256Single(vVolumesInt);
            
            // Multiply price * volume
            Vector256<float> priceVolume = Avx.Multiply(vPrices, vVolumes);
            
            sumPriceVolume = Avx.Add(sumPriceVolume, priceVolume);
            sumVolume = Avx.Add(sumVolume, vVolumes);
        }
        
        // Horizontal sum
        float totalPriceVolume = 0;
        float totalVolume = 0;
        
        for (int j = 0; j < 8; j++)
        {
            totalPriceVolume += sumPriceVolume.GetElement(j);
            totalVolume += sumVolume.GetElement(j);
        }
        
        // Handle remainder
        for (; i < count; i++)
        {
            totalPriceVolume += prices[i] * volumes[i];
            totalVolume += volumes[i];
        }
        
        return (decimal)(totalPriceVolume / totalVolume);
    }
    
    private unsafe decimal CalculateVWAPScalar(float* prices, int* volumes, int count)
    {
        float totalPriceVolume = 0;
        float totalVolume = 0;
        
        for (int i = 0; i < count; i++)
        {
            totalPriceVolume += prices[i] * volumes[i];
            totalVolume += volumes[i];
        }
        
        return (decimal)(totalPriceVolume / totalVolume);
    }
}
```

## Feature Detection Pattern

```csharp
public static class HardwareCapabilities
{
    public static readonly bool HasSSE = Sse.IsSupported;
    public static readonly bool HasSSE2 = Sse2.IsSupported;
    public static readonly bool HasSSE3 = Sse3.IsSupported;
    public static readonly bool HasSSE41 = Sse41.IsSupported;
    public static readonly bool HasSSE42 = Sse42.IsSupported;
    public static readonly bool HasAVX = Avx.IsSupported;
    public static readonly bool HasAVX2 = Avx2.IsSupported;
    public static readonly bool HasFMA = Fma.IsSupported;
    public static readonly bool HasAES = Aes.IsSupported;
    public static readonly bool HasPOPCNT = Popcnt.IsSupported;
    public static readonly bool HasBMI1 = Bmi1.IsSupported;
    public static readonly bool HasBMI2 = Bmi2.IsSupported;
    public static readonly bool HasLZCNT = Lzcnt.IsSupported;
    
    public static void PrintCapabilities()
    {
        Console.WriteLine($"SSE: {HasSSE}");
        Console.WriteLine($"SSE2: {HasSSE2}");
        Console.WriteLine($"AVX: {HasAVX}");
        Console.WriteLine($"AVX2: {HasAVX2}");
        Console.WriteLine($"FMA: {HasFMA}");
        Console.WriteLine($"AES: {HasAES}");
        Console.WriteLine($"POPCNT: {HasPOPCNT}");
        Console.WriteLine($"BMI1/BMI2: {HasBMI1}/{HasBMI2}");
    }
}
```

## Best Practices

### 1. Always Provide Fallbacks

```csharp
// ✅ Detect and fallback
public static void Process(float[] data)
{
    if (Avx2.IsSupported)
    {
        ProcessAVX2(data);
    }
    else if (Sse2.IsSupported)
    {
        ProcessSSE2(data);
    }
    else
    {
        ProcessScalar(data);
    }
}
```

### 2. Align Data for Best Performance

```csharp
// ✅ Aligned allocations
[StructLayout(LayoutKind.Sequential, Pack = 32)]
public struct AlignedData
{
    // Fields aligned to 32-byte boundary
}
```

### 3. Use Unsafe for Direct Memory Access

```csharp
// ✅ Fixed pointers for intrinsics
unsafe
{
    fixed (float* ptr = array)
    {
        Vector256<float> v = Avx.LoadVector256(ptr);
    }
}
```

### 4. Benchmark on Target Hardware

```csharp
// Different CPUs have different capabilities
// Always benchmark on production hardware
```

## Performance Characteristics

- **SIMD**: 4-16x throughput for vector operations
- **POPCNT**: 32x faster than loop
- **LZCNT/TZCNT**: 64x faster than loop
- **AES**: 10-100x faster than software
- **CRC32**: 10x faster than software

## When to Use

- High-throughput numeric processing
- Cryptographic operations
- Checksums and hashing
- Bit manipulation in hot paths
- Vector/matrix operations
- String processing (SSE4.2)

## When NOT to Use

- Portable code for unknown platforms
- Simple operations (overhead exceeds benefit)
- Complex branching logic
- Non-contiguous data

## Related Patterns

- [SIMD Vectorization](./simd-vectorization.md) — High-level SIMD
- [Unsafe Code Patterns](./unsafe-code-patterns.md) — Unsafe context
- [Bit Manipulation Techniques](./bit-manipulation-techniques.md) — Bit operations
- [Zero-Copy Techniques](./zero-copy-techniques.md) — Memory access

## References

- "Intel Intrinsics Guide" - Intel
- "Hardware Intrinsics in .NET" - Microsoft Docs
- "x86-64 Instruction Set Reference"
- "Agner Fog's Optimization Manuals"
