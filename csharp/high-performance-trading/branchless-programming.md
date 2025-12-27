# Branchless Programming

> Eliminate conditional branches to prevent CPU pipeline stalls—use arithmetic, bit manipulation, and predication for predictable, constant-time execution.

## Problem

Conditional branches (`if`, `switch`) cause CPU pipeline stalls when branch prediction fails. Modern CPUs speculatively execute code, but mispredicted branches require pipeline flushes—costing 10-20 cycles. In high-frequency systems processing millions of operations per second, branch mispredictions accumulate into significant latency.

## Example

### ❌ Before (Branching Code)

```csharp
public class BranchingProcessor
{
    // ❌ Branch causes pipeline stall if unpredictable
    public int Absolute(int value)
    {
        if (value < 0)
            return -value;
        else
            return value;
        
        // Branch misprediction: ~15 cycle penalty
    }
    
    // ❌ Unpredictable branch in hot loop
    public void ClampValues(Span<int> values, int min, int max)
    {
        for (int i = 0; i < values.Length; i++)
        {
            if (values[i] < min)        // Branch 1
                values[i] = min;
            else if (values[i] > max)   // Branch 2
                values[i] = max;
        }
        
        // 2 branches per iteration, likely mispredictions
    }
    
    // ❌ Branch in price comparison
    public decimal GetBetterPrice(decimal price1, decimal price2, bool isBuy)
    {
        if (isBuy)
            return price1 < price2 ? price1 : price2;  // Lower is better
        else
            return price1 > price2 ? price1 : price2;  // Higher is better
        
        // 2 branches per call
    }
}
```

### ✅ After (Branchless Code)

```csharp
public class BranchlessProcessor
{
    // ✅ Branchless absolute value
    public int Absolute(int value)
    {
        int mask = value >> 31;  // -1 if negative, 0 if positive
        return (value + mask) ^ mask;
        
        // Always 3 instructions, no branches!
    }
    
    // ✅ Branchless clamp
    public void ClampValues(Span<int> values, int min, int max)
    {
        for (int i = 0; i < values.Length; i++)
        {
            // Clamp using arithmetic (no branches)
            int v = values[i];
            v = v < min ? min : v;  // Compiler may use CMOV
            v = v > max ? max : v;
            values[i] = v;
        }
        
        // Or fully branchless:
        for (int i = 0; i < values.Length; i++)
        {
            int v = values[i];
            int tooLow = (min - v) >> 31;  // -1 if v < min, else 0
            v = (v & ~tooLow) | (min & tooLow);
            
            int tooHigh = (v - max - 1) >> 31;  // -1 if v <= max, else 0
            values[i] = (v & tooHigh) | (max & ~tooHigh);
        }
    }
    
    // ✅ Branchless price selection
    public decimal GetBetterPrice(decimal price1, decimal price2, bool isBuy)
    {
        // Convert bool to 0 or -1
        int buyMask = isBuy ? -1 : 0;
        
        bool p1Better = isBuy ? price1 < price2 : price1 > price2;
        
        // Use mask to select price
        return p1Better ? price1 : price2;
        
        // Or fully arithmetic (for integers):
        // int mask = (int)(price1 - price2) >> 31;
        // return (price1 & mask) | (price2 & ~mask);
    }
}
```

## Branchless Min/Max

```csharp
public static class BranchlessMinMax
{
    // ✅ Branchless min for integers
    public static int Min(int a, int b)
    {
        // a < b ? a : b without branch
        int diff = a - b;
        int mask = diff >> 31;  // -1 if a < b, else 0
        return b + (diff & mask);
    }
    
    // ✅ Branchless max for integers
    public static int Max(int a, int b)
    {
        int diff = a - b;
        int mask = diff >> 31;
        return a - (diff & mask);
    }
    
    // ✅ Using bit tricks
    public static int MinBitTrick(int a, int b)
    {
        return b ^ ((a ^ b) & -(a < b ? 1 : 0));
    }
    
    // ✅ Branchless for floats (use compiler intrinsics)
    public static float MinFloat(float a, float b)
    {
        // Compiler generates MINSS instruction (branchless)
        return Math.Min(a, b);
    }
}
```

## Branchless Selection (CMOV pattern)

```csharp
public static class BranchlessSelection
{
    // ✅ Conditional move pattern
    public static int Select(bool condition, int ifTrue, int ifFalse)
    {
        // Compiles to CMOV instruction
        return condition ? ifTrue : ifFalse;
    }
    
    // ✅ Using arithmetic mask
    public static int SelectMask(bool condition, int ifTrue, int ifFalse)
    {
        int mask = condition ? -1 : 0;
        return (ifTrue & mask) | (ifFalse & ~mask);
    }
    
    // ✅ Select from array branchlessly
    public static T SelectFromArray<T>(T[] array, int index, bool condition)
    {
        int actualIndex = condition ? index : 0;
        return array[actualIndex];
    }
}
```

## Branchless Sign and Comparison

```csharp
public static class BranchlessComparison
{
    // ✅ Get sign without branch (-1, 0, or 1)
    public static int Sign(int value)
    {
        return (value >> 31) | (int)((uint)-value >> 31);
    }
    
    // ✅ Compare without branch (returns -1, 0, or 1)
    public static int Compare(int a, int b)
    {
        return Sign(a - b);
    }
    
    // ✅ Branchless equality check (returns 0 or 1)
    public static int Equal(int a, int b)
    {
        int diff = a ^ b;
        return (int)(((uint)diff | (uint)-diff) >> 31) ^ 1;
    }
    
    // ✅ Not equal
    public static int NotEqual(int a, int b)
    {
        int diff = a ^ b;
        return (int)(((uint)diff | (uint)-diff) >> 31);
    }
    
    // ✅ Less than
    public static int LessThan(int a, int b)
    {
        return (a - b) >> 31;  // Returns -1 if a < b
    }
}
```

## Branchless Array Operations

```csharp
public class BranchlessArrayOps
{
    // ✅ Conditional increment without branch
    public void IncrementIf(Span<int> values, Span<bool> conditions)
    {
        for (int i = 0; i < values.Length; i++)
        {
            int mask = conditions[i] ? 1 : 0;
            values[i] += mask;  // Branchless increment
        }
    }
    
    // ✅ Conditional set without branch
    public void SetIf(Span<int> values, int setValue, Span<bool> conditions)
    {
        for (int i = 0; i < values.Length; i++)
        {
            int mask = conditions[i] ? -1 : 0;
            values[i] = (setValue & mask) | (values[i] & ~mask);
        }
    }
    
    // ✅ Count matching elements without branch
    public int CountMatching(ReadOnlySpan<int> values, int target)
    {
        int count = 0;
        
        for (int i = 0; i < values.Length; i++)
        {
            int equal = Equal(values[i], target);
            count += equal;  // Add 1 if equal, 0 otherwise
        }
        
        return count;
    }
    
    private static int Equal(int a, int b)
    {
        int diff = a ^ b;
        return (int)(((uint)diff | (uint)-diff) >> 31) ^ 1;
    }
}
```

## Branchless State Machines

```csharp
public class BranchlessStateMachine
{
    // ✅ State transition without branches
    private static readonly int[,] transitions = new int[4, 4]
    {
        // Current state → Next state based on input (0-3)
        { 0, 1, 2, 0 },  // State 0
        { 1, 2, 3, 0 },  // State 1
        { 2, 3, 0, 1 },  // State 2
        { 3, 0, 1, 2 }   // State 3
    };
    
    private int state;
    
    public void ProcessInput(int input)
    {
        // No branches, just array lookup
        state = transitions[state, input];
    }
    
    // ✅ Action dispatch without branches
    private static readonly Action<int>[] actions = new Action<int>[]
    {
        HandleState0,
        HandleState1,
        HandleState2,
        HandleState3
    };
    
    public void Execute(int value)
    {
        actions[state](value);  // Indirect call, no branches
    }
    
    private static void HandleState0(int value) { }
    private static void HandleState1(int value) { }
    private static void HandleState2(int value) { }
    private static void HandleState3(int value) { }
}
```

## Branchless Validation

```csharp
public class BranchlessValidation
{
    // ✅ Range check without branch
    public static bool InRange(int value, int min, int max)
    {
        // Single comparison using unsigned arithmetic
        return (uint)(value - min) <= (uint)(max - min);
    }
    
    // ✅ Multiple range checks
    public static bool InRanges(int value, int min1, int max1, int min2, int max2)
    {
        bool range1 = (uint)(value - min1) <= (uint)(max1 - min1);
        bool range2 = (uint)(value - min2) <= (uint)(max2 - min2);
        return range1 | range2;  // Bitwise OR avoids branch
    }
    
    // ✅ Validate flags branchlessly
    public static bool HasAllFlags(int value, int required)
    {
        return (value & required) == required;
    }
}
```

## Order Book Branchless Updates

```csharp
public class BranchlessOrderBook
{
    // ✅ Update best bid/ask without branches
    public void UpdateBest(ref decimal currentBest, decimal newPrice, bool isBid)
    {
        // For bid: want max; for ask: want min
        // Use arithmetic to avoid branches
        
        bool isNewBetter = isBid 
            ? newPrice > currentBest 
            : newPrice < currentBest;
        
        // Branchless update using CMOV
        currentBest = isNewBetter ? newPrice : currentBest;
    }
    
    // ✅ Process tick branchlessly
    public void ProcessTick(decimal price, int size, bool isBuy, ref long buyVolume, ref long sellVolume)
    {
        // Branchless volume update
        int buyMask = isBuy ? 1 : 0;
        int sellMask = isBuy ? 0 : 1;
        
        buyVolume += size * buyMask;
        sellVolume += size * sellMask;
    }
}
```

## Best Practices

### 1. Profile Branch Mispredictions

```csharp
// Use performance counters to measure:
// - branch-misses
// - branch-instructions
// - Misprediction rate > 5% = consider branchless
```

### 2. Know When Branches Are OK

```csharp
// ✅ Predictable branches are fine
for (int i = 0; i < array.Length; i++)  // Loop branch: predicted
{
    // ...
}

// ✅ Early exits are fine
if (array == null) return;  // Rarely taken

// ❌ Unpredictable data-dependent branches
if (prices[i] > threshold)  // Data-dependent: mispredicts
```

### 3. Use Compiler Intrinsics

```csharp
// ✅ Let compiler generate CMOV
int result = condition ? a : b;  // Compiles to CMOV

// ✅ Math.Min/Max often branchless
int min = Math.Min(a, b);  // May use CMOV/MINSS
```

### 4. Don't Over-Optimize

```csharp
// ❌ Premature branchless optimization
public int ComplexLogic(int a, int b, int c)
{
    // Unreadable bit manipulation...
}

// ✅ Profile first, optimize hot paths only
public int SimpleLogic(int a, int b, int c)
{
    if (a > b) return c;  // Fine if rarely mispredicted
    return a + b;
}
```

## Performance Characteristics

- **Pipeline stalls**: Eliminated
- **Predictable latency**: Every execution same cost
- **Best case**: 2-5x speedup with unpredictable branches
- **Worst case**: 10-20% slower than perfectly predicted branch

## When to Use

- Hot paths with >1% branch misprediction rate
- Data-dependent branches in tight loops
- Constant-time requirements (security)
- Financial calculations (tick processing)
- Comparison and selection operations

## When NOT to Use

- Complex business logic (readability matters)
- Rarely executed code
- Branches with >95% prediction accuracy
- When code becomes unmaintainable

## Related Patterns

- [Bit Manipulation Techniques](./bit-manipulation-techniques.md) — Bit tricks
- [SIMD Vectorization](./simd-vectorization.md) — Vectorized selection
- [Constant Time Operations](./constant-time-operations.md) — Security
- [Hot Path Optimization](./hot-path-optimization.md) — Critical paths

## References

- "Computer Architecture: A Quantitative Approach" by Hennessy & Patterson
- "Branchless Programming in C++" by Fedor Pikus
- "Intel® 64 and IA-32 Architectures Optimization Reference Manual"
- "What Every Programmer Should Know About Branch Prediction"
