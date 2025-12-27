# SIMD Vectorization

> Use SIMD (Single Instruction, Multiple Data) to process multiple data elements in parallel—leverage CPU vector instructions for massive throughput gains in data-parallel workloads.

## Problem

Processing arrays element-by-element is inefficient on modern CPUs. Modern processors have 128-bit, 256-bit, or 512-bit vector registers that can process 4, 8, or 16 elements simultaneously. Scalar code wastes this parallel processing capability, leaving performance on the table.

## Example

### ❌ Before (Scalar Processing)

```csharp
public class ScalarProcessor
{
    public void AddArrays(float[] a, float[] b, float[] result)
    {
        // Processes one element per iteration
        for (int i = 0; i < a.Length; i++)
        {
            result[i] = a[i] + b[i];
        }
        
        // CPU executes ~100M operations/sec
        // Only uses scalar ALU, vector units idle
    }
    
    public void ScaleArray(float[] data, float scale)
    {
        for (int i = 0; i < data.Length; i++)
        {
            data[i] *= scale;
        }
        
        // 1 operation per cycle per core
    }
}
```

### ✅ After (SIMD with Vector<T>)

```csharp
public class VectorizedProcessor
{
    public void AddArrays(float[] a, float[] b, float[] result)
    {
        int vectorSize = Vector<float>.Count;  // 4, 8, or 16 depending on CPU
        int i = 0;
        
        // Process in vector-sized chunks
        for (; i <= a.Length - vectorSize; i += vectorSize)
        {
            var va = new Vector<float>(a, i);
            var vb = new Vector<float>(b, i);
            var vr = va + vb;  // Adds 4-16 floats in one instruction!
            vr.CopyTo(result, i);
        }
        
        // Handle remaining elements
        for (; i < a.Length; i++)
        {
            result[i] = a[i] + b[i];
        }
        
        // 4-16x throughput improvement!
    }
    
    public void ScaleArray(float[] data, float scale)
    {
        var vectorScale = new Vector<float>(scale);
        int vectorSize = Vector<float>.Count;
        int i = 0;
        
        for (; i <= data.Length - vectorSize; i += vectorSize)
        {
            var v = new Vector<float>(data, i);
            v = v * vectorScale;  // Multiplies 4-16 values simultaneously
            v.CopyTo(data, i);
        }
        
        for (; i < data.Length; i++)
        {
            data[i] *= scale;
        }
    }
}
```

## SIMD Operations

```csharp
public static class SIMDOperations
{
    // ✅ Dot product with SIMD
    public static float DotProduct(ReadOnlySpan<float> a, ReadOnlySpan<float> b)
    {
        float sum = 0;
        int vectorSize = Vector<float>.Count;
        var sumVector = Vector<float>.Zero;
        int i = 0;
        
        // Vectorized accumulation
        for (; i <= a.Length - vectorSize; i += vectorSize)
        {
            var va = new Vector<float>(a.Slice(i, vectorSize));
            var vb = new Vector<float>(b.Slice(i, vectorSize));
            sumVector += va * vb;
        }
        
        // Sum vector lanes
        for (int j = 0; j < vectorSize; j++)
        {
            sum += sumVector[j];
        }
        
        // Handle remainder
        for (; i < a.Length; i++)
        {
            sum += a[i] * b[i];
        }
        
        return sum;
    }
    
    // ✅ Find maximum with SIMD
    public static float FindMax(ReadOnlySpan<float> data)
    {
        int vectorSize = Vector<float>.Count;
        var maxVector = new Vector<float>(float.MinValue);
        int i = 0;
        
        for (; i <= data.Length - vectorSize; i += vectorSize)
        {
            var v = new Vector<float>(data.Slice(i, vectorSize));
            maxVector = Vector.Max(maxVector, v);
        }
        
        float max = float.MinValue;
        for (int j = 0; j < vectorSize; j++)
        {
            max = Math.Max(max, maxVector[j]);
        }
        
        for (; i < data.Length; i++)
        {
            max = Math.Max(max, data[i]);
        }
        
        return max;
    }
    
    // ✅ Conditional operations (masks)
    public static void ClampArray(Span<float> data, float min, float max)
    {
        int vectorSize = Vector<float>.Count;
        var minVector = new Vector<float>(min);
        var maxVector = new Vector<float>(max);
        int i = 0;
        
        for (; i <= data.Length - vectorSize; i += vectorSize)
        {
            var v = new Vector<float>(data.Slice(i, vectorSize));
            v = Vector.Max(v, minVector);  // Clamp to min
            v = Vector.Min(v, maxVector);  // Clamp to max
            v.CopyTo(data.Slice(i, vectorSize));
        }
        
        for (; i < data.Length; i++)
        {
            data[i] = Math.Max(min, Math.Min(max, data[i]));
        }
    }
}
```

## Hardware Intrinsics (Advanced)

```csharp
using System.Runtime.Intrinsics;
using System.Runtime.Intrinsics.X86;

public static class IntrinsicsOperations
{
    // ✅ AVX2 for 256-bit operations
    public static void AddArraysAVX2(float[] a, float[] b, float[] result)
    {
        if (!Avx2.IsSupported)
        {
            throw new NotSupportedException("AVX2 not available");
        }
        
        int i = 0;
        
        unsafe
        {
            fixed (float* pa = a, pb = b, pr = result)
            {
                // Process 8 floats at a time (256-bit / 32-bit)
                for (; i <= a.Length - 8; i += 8)
                {
                    var va = Avx.LoadVector256(pa + i);
                    var vb = Avx.LoadVector256(pb + i);
                    var vr = Avx.Add(va, vb);
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
    
    // ✅ Fused multiply-add (FMA)
    public static void MultiplyAdd(float[] a, float[] b, float[] c, float[] result)
    {
        if (!Fma.IsSupported)
        {
            throw new NotSupportedException("FMA not available");
        }
        
        int i = 0;
        
        unsafe
        {
            fixed (float* pa = a, pb = b, pc = c, pr = result)
            {
                for (; i <= a.Length - 8; i += 8)
                {
                    var va = Avx.LoadVector256(pa + i);
                    var vb = Avx.LoadVector256(pb + i);
                    var vc = Avx.LoadVector256(pc + i);
                    
                    // result = (a * b) + c in single instruction!
                    var vr = Fma.MultiplyAdd(va, vb, vc);
                    Avx.Store(pr + i, vr);
                }
            }
        }
        
        for (; i < a.Length; i++)
        {
            result[i] = (a[i] * b[i]) + c[i];
        }
    }
}
```

## Market Data Processing with SIMD

```csharp
public readonly record struct Tick(float Price, int Volume);

public class TickProcessor
{
    // ✅ Calculate VWAP using SIMD
    public float CalculateVWAP(ReadOnlySpan<Tick> ticks)
    {
        int vectorSize = Vector<float>.Count;
        var priceVolumeSum = Vector<float>.Zero;
        var volumeSum = Vector<float>.Zero;
        
        int i = 0;
        Span<float> prices = stackalloc float[vectorSize];
        Span<float> volumes = stackalloc float[vectorSize];
        
        for (; i <= ticks.Length - vectorSize; i += vectorSize)
        {
            // Gather data into contiguous arrays
            for (int j = 0; j < vectorSize; j++)
            {
                prices[j] = ticks[i + j].Price;
                volumes[j] = ticks[i + j].Volume;
            }
            
            var vPrices = new Vector<float>(prices);
            var vVolumes = new Vector<float>(volumes);
            
            priceVolumeSum += vPrices * vVolumes;
            volumeSum += vVolumes;
        }
        
        // Reduce vectors to scalars
        float totalPriceVolume = 0;
        float totalVolume = 0;
        
        for (int j = 0; j < vectorSize; j++)
        {
            totalPriceVolume += priceVolumeSum[j];
            totalVolume += volumeSum[j];
        }
        
        // Handle remainder
        for (; i < ticks.Length; i++)
        {
            totalPriceVolume += ticks[i].Price * ticks[i].Volume;
            totalVolume += ticks[i].Volume;
        }
        
        return totalPriceVolume / totalVolume;
    }
}
```

## Moving Average with SIMD

```csharp
public class SIMDMovingAverage
{
    // ✅ Simple moving average with vectorization
    public void CalculateSMA(ReadOnlySpan<float> prices, Span<float> sma, int period)
    {
        if (prices.Length < period)
            return;
        
        int vectorSize = Vector<float>.Count;
        
        for (int i = 0; i < prices.Length - period + 1; i++)
        {
            var sum = Vector<float>.Zero;
            int j = 0;
            
            // Sum using SIMD
            for (; j <= period - vectorSize; j += vectorSize)
            {
                var v = new Vector<float>(prices.Slice(i + j, vectorSize));
                sum += v;
            }
            
            float scalarSum = 0;
            for (int k = 0; k < vectorSize; k++)
            {
                scalarSum += sum[k];
            }
            
            for (; j < period; j++)
            {
                scalarSum += prices[i + j];
            }
            
            sma[i] = scalarSum / period;
        }
    }
}
```

## Best Practices

### 1. Check Hardware Support

```csharp
// ✅ Check before using SIMD
if (Vector.IsHardwareAccelerated)
{
    ProcessWithSIMD(data);
}
else
{
    ProcessScalar(data);
}

// ✅ Check specific instruction sets
if (Avx2.IsSupported)
{
    ProcessWithAVX2(data);
}
else if (Sse2.IsSupported)
{
    ProcessWithSSE2(data);
}
```

### 2. Align Data When Possible

```csharp
// ✅ Prefer aligned allocations
[StructLayout(LayoutKind.Sequential, Pack = 32)]
public struct AlignedData
{
    public float Value1;
    public float Value2;
    // ... more fields
}
```

### 3. Handle Remainder Elements

```csharp
// ✅ Always handle non-vector-aligned tail
int vectorSize = Vector<float>.Count;
int i = 0;

// Vectorized loop
for (; i <= array.Length - vectorSize; i += vectorSize)
{
    ProcessVector(array, i);
}

// Scalar tail
for (; i < array.Length; i++)
{
    ProcessScalar(array[i]);
}
```

### 4. Minimize Gather/Scatter

```csharp
// ❌ Inefficient: non-contiguous access
for (int i = 0; i < structs.Length; i += vectorSize)
{
    // Gathering scattered fields is slow
    var prices = new Vector<float>(structs.Select(s => s.Price).ToArray(), i);
}

// ✅ Better: use SoA (Structure of Arrays)
public class MarketData
{
    public float[] Prices;
    public int[] Volumes;
    
    // Now prices are contiguous, perfect for SIMD!
}
```

## Performance Characteristics

- **Throughput**: 4-16x improvement for float/int operations
- **Latency**: Same per operation, but processes multiple elements
- **Best for**: Large arrays (>100 elements), numeric operations
- **Cache**: Excellent cache utilization with contiguous access

## When to Use

- Processing large numeric arrays (prices, volumes, signals)
- Mathematical operations on vectors/matrices
- Image/signal processing
- Data transformations and aggregations
- Calculating indicators (moving averages, RSI, etc.)

## When NOT to Use

- Small arrays (<100 elements) — overhead exceeds benefit
- Non-numeric data or complex objects
- Operations with heavy branching
- Irregular memory access patterns

## Related Patterns

- [Branchless Programming](./branchless-programming.md) — Works well with SIMD
- [Packed Data Structures](./packed-data-structures.md) — SoA for SIMD
- [Hot Path Optimization](./hot-path-optimization.md) — SIMD in critical paths
- [Zero-Copy Techniques](./zero-copy-techniques.md) — Span with SIMD

## References

- ".NET SIMD Support" - Microsoft Docs
- "Hardware Intrinsics in .NET" - .NET Blog
- "Intel Intrinsics Guide" - Intel Documentation
- "Writing High-Performance .NET Code" by Ben Watson
