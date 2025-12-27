# Bit Manipulation Techniques

> Use bit-level operations for compact storage, fast arithmetic, and zero-overhead flags—leverage CPU's native bit operations for maximum performance.

## Problem

Traditional approaches using booleans, enums, and separate fields waste memory and CPU cycles. Modern CPUs have dedicated bit manipulation instructions that execute in a single cycle. Using higher-level abstractions misses these optimization opportunities and produces bloated data structures.

## Example

### ❌ Before (Separate Fields)

```csharp
public class OrderFlags
{
    public bool IsBuyOrder;
    public bool IsMarketOrder;
    public bool IsIOC;
    public bool IsFillOrKill;
    public bool IsHidden;
    public bool IsPostOnly;
    // 6 bytes minimum (+ padding = 8-16 bytes)
    
    public void SetBuyOrder() => IsBuyOrder = true;
    public bool IsBuy() => IsBuyOrder;
}

// Wastes memory, slower to copy, poor cache utilization
```

### ✅ After (Bit Flags)

```csharp
[Flags]
public enum OrderFlags : ulong
{
    None = 0,
    Buy = 1 << 0,          // 0x0001
    Market = 1 << 1,       // 0x0002
    IOC = 1 << 2,          // 0x0004
    FillOrKill = 1 << 3,   // 0x0008
    Hidden = 1 << 4,       // 0x0010
    PostOnly = 1 << 5,     // 0x0020
    // Can hold 64 flags in 8 bytes!
}

public readonly record struct Order(
    long OrderId,
    OrderFlags Flags,
    decimal Price,
    int Quantity)
{
    // ✅ Single-cycle bit operations
    public bool IsBuy() => (Flags & OrderFlags.Buy) != 0;
    public bool IsMarket() => (Flags & OrderFlags.Market) != 0;
    public bool HasFlags(OrderFlags flags) => (Flags & flags) == flags;
    
    // ✅ Combine flags efficiently
    public Order WithFlags(OrderFlags additionalFlags)
        => this with { Flags = Flags | additionalFlags };
    
    public Order ClearFlags(OrderFlags flagsToClear)
        => this with { Flags = Flags & ~flagsToClear };
}
```

## Common Bit Operations

```csharp
public static class BitOperations
{
    // ✅ Set bit
    public static int SetBit(int value, int position)
    {
        return value | (1 << position);
    }
    
    // ✅ Clear bit
    public static int ClearBit(int value, int position)
    {
        return value & ~(1 << position);
    }
    
    // ✅ Toggle bit
    public static int ToggleBit(int value, int position)
    {
        return value ^ (1 << position);
    }
    
    // ✅ Test bit
    public static bool TestBit(int value, int position)
    {
        return (value & (1 << position)) != 0;
    }
    
    // ✅ Count set bits (population count)
    public static int PopCount(uint value)
    {
        return System.Numerics.BitOperations.PopCount(value);
    }
    
    // ✅ Find first set bit
    public static int TrailingZeroCount(uint value)
    {
        return System.Numerics.BitOperations.TrailingZeroCount(value);
    }
    
    // ✅ Find last set bit
    public static int LeadingZeroCount(uint value)
    {
        return System.Numerics.BitOperations.LeadingZeroCount(value);
    }
    
    // ✅ Round up to power of 2
    public static uint RoundUpToPowerOf2(uint value)
    {
        return System.Numerics.BitOperations.RoundUpToPowerOf2(value);
    }
    
    // ✅ Check if power of 2
    public static bool IsPowerOfTwo(uint value)
    {
        return System.Numerics.BitOperations.IsPowerOfTwo(value);
    }
}
```

## Packed Bit Fields

```csharp
// ✅ Pack multiple values into single integer
public readonly struct PackedOrderInfo
{
    private readonly uint packed;
    
    // Layout: [24 bits: OrderId][4 bits: OrderType][4 bits: Flags]
    
    private const int OrderIdBits = 24;
    private const int OrderTypeBits = 4;
    private const int FlagsBits = 4;
    
    private const uint OrderIdMask = (1u << OrderIdBits) - 1;  // 0xFFFFFF
    private const uint OrderTypeMask = (1u << OrderTypeBits) - 1;  // 0xF
    private const uint FlagsMask = (1u << FlagsBits) - 1;  // 0xF
    
    public PackedOrderInfo(uint orderId, byte orderType, byte flags)
    {
        packed = (orderId & OrderIdMask) |
                 ((uint)(orderType & OrderTypeMask) << OrderIdBits) |
                 ((uint)(flags & FlagsMask) << (OrderIdBits + OrderTypeBits));
    }
    
    public uint OrderId => packed & OrderIdMask;
    public byte OrderType => (byte)((packed >> OrderIdBits) & OrderTypeMask);
    public byte Flags => (byte)((packed >> (OrderIdBits + OrderTypeBits)) & FlagsMask);
    
    // 4 bytes instead of 12+ bytes!
}
```

## Fast Integer Math with Bits

```csharp
public static class BitMath
{
    // ✅ Fast multiply by power of 2
    public static int MultiplyByPowerOf2(int value, int power)
    {
        return value << power;  // x * 2^n = x << n
    }
    
    // ✅ Fast divide by power of 2
    public static int DivideByPowerOf2(int value, int power)
    {
        return value >> power;  // x / 2^n = x >> n
    }
    
    // ✅ Fast modulo power of 2
    public static int ModuloPowerOf2(int value, int powerOf2)
    {
        return value & (powerOf2 - 1);  // x % 2^n = x & (2^n - 1)
    }
    
    // ✅ Check if even/odd
    public static bool IsEven(int value) => (value & 1) == 0;
    public static bool IsOdd(int value) => (value & 1) != 0;
    
    // ✅ Swap without temp variable
    public static void Swap(ref int a, ref int b)
    {
        a ^= b;
        b ^= a;
        a ^= b;
    }
    
    // ✅ Absolute value without branching
    public static int Abs(int value)
    {
        int mask = value >> 31;  // -1 if negative, 0 if positive
        return (value + mask) ^ mask;
    }
    
    // ✅ Min without branching
    public static int Min(int a, int b)
    {
        return b ^ ((a ^ b) & -(a < b ? 1 : 0));
    }
    
    // ✅ Max without branching
    public static int Max(int a, int b)
    {
        return a ^ ((a ^ b) & -(a < b ? 1 : 0));
    }
    
    // ✅ Sign function (-1, 0, or 1)
    public static int Sign(int value)
    {
        return (value >> 31) | (int)((uint)-value >> 31);
    }
}
```

## Bit Scanning for Market Data

```csharp
public class OrderBook
{
    // ✅ Use bit flags for price level tracking
    private ulong activeLevelsBitmap;  // Track 64 price levels
    private decimal[] priceLevels = new decimal[64];
    
    public void AddPriceLevel(int level)
    {
        activeLevelsBitmap |= (1ul << level);
    }
    
    public void RemovePriceLevel(int level)
    {
        activeLevelsBitmap &= ~(1ul << level);
    }
    
    public bool HasPriceLevel(int level)
    {
        return (activeLevelsBitmap & (1ul << level)) != 0;
    }
    
    // ✅ Find best bid (highest set bit)
    public int GetBestBidLevel()
    {
        if (activeLevelsBitmap == 0)
            return -1;
        
        return 63 - System.Numerics.BitOperations.LeadingZeroCount(activeLevelsBitmap);
    }
    
    // ✅ Find best ask (lowest set bit)
    public int GetBestAskLevel()
    {
        if (activeLevelsBitmap == 0)
            return -1;
        
        return System.Numerics.BitOperations.TrailingZeroCount(activeLevelsBitmap);
    }
    
    // ✅ Count active levels
    public int GetActiveLevelCount()
    {
        return System.Numerics.BitOperations.PopCount(activeLevelsBitmap);
    }
    
    // ✅ Iterate over active levels (fast)
    public void ProcessActiveLevels(Action<int> processLevel)
    {
        ulong bitmap = activeLevelsBitmap;
        
        while (bitmap != 0)
        {
            int level = System.Numerics.BitOperations.TrailingZeroCount(bitmap);
            processLevel(level);
            bitmap &= bitmap - 1;  // Clear lowest set bit
        }
    }
}
```

## Bit Masks for Permissions

```csharp
[Flags]
public enum TradingPermissions : uint
{
    None = 0,
    ViewMarketData = 1 << 0,
    PlaceOrders = 1 << 1,
    CancelOrders = 1 << 2,
    ViewPositions = 1 << 3,
    ModifyOrders = 1 << 4,
    TradeEquities = 1 << 5,
    TradeOptions = 1 << 6,
    TradeFutures = 1 << 7,
    // ... up to 32 permissions
}

public readonly record struct TradingAccount(
    string AccountId,
    TradingPermissions Permissions)
{
    // ✅ Fast permission checks (single instruction)
    public bool CanPlaceOrder()
        => (Permissions & TradingPermissions.PlaceOrders) != 0;
    
    public bool HasAllPermissions(TradingPermissions required)
        => (Permissions & required) == required;
    
    public bool HasAnyPermission(TradingPermissions options)
        => (Permissions & options) != 0;
    
    // ✅ Grant permissions
    public TradingAccount GrantPermissions(TradingPermissions toGrant)
        => this with { Permissions = Permissions | toGrant };
    
    // ✅ Revoke permissions
    public TradingAccount RevokePermissions(TradingPermissions toRevoke)
        => this with { Permissions = Permissions & ~toRevoke };
}
```

## Compact State Machines with Bits

```csharp
public readonly struct OrderState
{
    private readonly byte state;
    
    // Pack state + flags into single byte
    // [3 bits: State][5 bits: Flags]
    private const byte StateMask = 0b0000_0111;
    private const byte FlagsMask = 0b1111_1000;
    
    private const byte PendingState = 0;
    private const byte ActiveState = 1;
    private const byte FilledState = 2;
    private const byte CancelledState = 3;
    
    private const byte PartiallyFilledFlag = 1 << 3;
    private const byte AmendedFlag = 1 << 4;
    
    public OrderState(byte state, bool partiallyFilled = false, bool amended = false)
    {
        this.state = (byte)(state & StateMask);
        
        if (partiallyFilled)
            this.state |= PartiallyFilledFlag;
        
        if (amended)
            this.state |= AmendedFlag;
    }
    
    public byte State => (byte)(state & StateMask);
    public bool IsPartiallyFilled => (state & PartiallyFilledFlag) != 0;
    public bool IsAmended => (state & AmendedFlag) != 0;
    
    // Single byte stores complex state!
}
```

## Best Practices

### 1. Use System.Numerics.BitOperations

```csharp
// ✅ Use built-in hardware-accelerated operations
using System.Numerics;

int count = BitOperations.PopCount(flags);
int firstBit = BitOperations.TrailingZeroCount(flags);
uint rounded = BitOperations.RoundUpToPowerOf2(size);
```

### 2. Document Bit Layouts

```csharp
// ✅ Clear documentation
/// <summary>
/// Packed order info: [24 bits: OrderId][4 bits: Type][4 bits: Flags]
/// </summary>
public readonly struct PackedOrder
{
    // Bit layout constants
    private const int OrderIdBits = 24;
    // ...
}
```

### 3. Use Flags Enum for Multiple Options

```csharp
// ✅ Flags enum for bit combinations
[Flags]
public enum Options
{
    None = 0,
    Option1 = 1 << 0,
    Option2 = 1 << 1,
    // ...
}
```

### 4. Avoid Over-Optimization

```csharp
// ❌ Don't pack everything just because you can
public struct OverlyPacked  // Hard to maintain!
{
    private ulong packed;  // 17 different fields packed...
}

// ✅ Pack only hot data structures
public struct TickData  // Used billions of times
{
    private uint packed;  // Worth it here
}
```

## Performance Characteristics

- **Bit operations**: 1 CPU cycle
- **Memory savings**: 8-16x for flags
- **Cache efficiency**: Improved (smaller data)
- **Branch-free**: Many operations avoid branches

## When to Use

- Multiple boolean flags (>3 flags)
- Compact data structures (ticks, events)
- Performance-critical code paths
- Permission systems
- State machines with few states

## When NOT to Use

- Complex business logic (use enums/classes)
- Rarely executed code
- When readability is more important
- Debugging-intensive code

## Related Patterns

- [Packed Data Structures](./packed-data-structures.md) — Dense layouts
- [Branchless Programming](./branchless-programming.md) — Bit tricks
- [Struct Layout](../patterns/struct-layout.md) — Memory layout
- [Hot Path Optimization](./hot-path-optimization.md) — Critical paths

## References

- "Bit Twiddling Hacks" by Sean Eron Anderson
- "Hacker's Delight" by Henry S. Warren Jr.
- ".NET BitOperations" - Microsoft Docs
- "Computer Architecture" by Hennessy & Patterson
