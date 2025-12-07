# Struct Layout (Memory and Cache Optimization)

> Structs with poor field ordering waste memory and cause cache misses—use explicit layout and field ordering for optimal performance.

## Problem

The CLR arranges struct fields in memory, but the default layout may not be optimal. Poor field ordering can waste memory through padding and harm cache performance by separating frequently-accessed fields.

## Example

### ❌ Before

```csharp
// Poorly ordered fields
public struct PlayerState
{
    public bool IsAlive;        // 1 byte  (offset 0)
                                // 7 bytes padding
    public double Health;       // 8 bytes (offset 8)
    public bool IsInvincible;   // 1 byte  (offset 16)
                                // 3 bytes padding
    public int Score;           // 4 bytes (offset 20)
    public bool HasPowerUp;     // 1 byte  (offset 24)
                                // 7 bytes padding
    public long Timestamp;      // 8 bytes (offset 32)
    
    // Total size: 40 bytes (15 bytes wasted on padding!)
}

// What the memory actually looks like:
// [IsAlive:1][padding:7][Health:8][IsInvincible:1][padding:3][Score:4][HasPowerUp:1][padding:7][Timestamp:8]
```

**Problems:**
- 40 bytes total, but only 25 bytes of actual data
- 15 bytes wasted on padding (37.5% overhead)
- Bools separated by large fields → cache misses when checking multiple flags
- More memory = fewer structs fit in cache

### ✅ After

```csharp
// Optimally ordered: largest to smallest
public struct PlayerState
{
    // 8-byte aligned fields first
    public double Health;       // 8 bytes (offset 0)
    public long Timestamp;      // 8 bytes (offset 8)
    
    // 4-byte aligned fields
    public int Score;           // 4 bytes (offset 16)
    
    // 1-byte fields together at the end
    public bool IsAlive;        // 1 byte  (offset 20)
    public bool IsInvincible;   // 1 byte  (offset 21)
    public bool HasPowerUp;     // 1 byte  (offset 22)
                                // 1 byte padding to align to 8-byte boundary
    
    // Total size: 24 bytes (only 1 byte of padding)
}

// Memory layout:
// [Health:8][Timestamp:8][Score:4][IsAlive:1][IsInvincible:1][HasPowerUp:1][padding:1]
```

**Improvements:**
- 24 bytes instead of 40 bytes (40% size reduction)
- Only 1 byte of padding instead of 15
- All bools are adjacent → better cache locality
- 66% more structs fit in L1 cache

## Explicit Layout

For maximum control:

```csharp
[StructLayout(LayoutKind.Explicit, Size = 24)]
public struct PlayerState
{
    [FieldOffset(0)]  public double Health;
    [FieldOffset(8)]  public long Timestamp;
    [FieldOffset(16)] public int Score;
    [FieldOffset(20)] public bool IsAlive;
    [FieldOffset(21)] public bool IsInvincible;
    [FieldOffset(22)] public bool HasPowerUp;
}

// Size is guaranteed to be 24 bytes
Console.WriteLine(Unsafe.SizeOf<PlayerState>());  // Output: 24
```

## Cache-Friendly Hot/Cold Splitting

Separate frequently-accessed fields from rarely-accessed fields:

```csharp
// Hot: accessed every frame
public struct ParticleHot
{
    public Vector3 Position;     // 12 bytes
    public Vector3 Velocity;     // 12 bytes
    public float TimeAlive;      // 4 bytes
    
    // Total: 28 bytes → fits in one cache line (64 bytes)
}

// Cold: accessed occasionally
public struct ParticleCold
{
    public Color Color;          // 16 bytes
    public ParticleType Type;    // 4 bytes
    public int Id;               // 4 bytes
    public string Name;          // 8 bytes (reference)
}

// Store separately for better cache utilization
public class ParticleSystem
{
    private ParticleHot[] _hotData;     // Tightly packed, cache-friendly
    private ParticleCold[] _coldData;   // Accessed rarely
    
    public void Update(float deltaTime)
    {
        // Update loop only touches hot data → excellent cache performance
        for (int i = 0; i < _hotData.Length; i++)
        {
            ref var particle = ref _hotData[i];
            particle.Position += particle.Velocity * deltaTime;
            particle.TimeAlive += deltaTime;
        }
        
        // Hot data is much more compact → more fits in cache
    }
}
```

## Benchmark Example

```csharp
[MemoryDiagnoser]
public class StructLayoutBenchmark
{
    private const int Count = 10_000;
    
    private PoorLayout[] _poorData;
    private OptimalLayout[] _optimalData;
    
    [GlobalSetup]
    public void Setup()
    {
        _poorData = new PoorLayout[Count];
        _optimalData = new OptimalLayout[Count];
        
        for (int i = 0; i < Count; i++)
        {
            _poorData[i] = new PoorLayout { Health = 100, IsAlive = true, Score = i };
            _optimalData[i] = new OptimalLayout { Health = 100, IsAlive = true, Score = i };
        }
    }
    
    [Benchmark(Baseline = true)]
    public long SumScores_PoorLayout()
    {
        long sum = 0;
        for (int i = 0; i < _poorData.Length; i++)
        {
            if (_poorData[i].IsAlive)
                sum += _poorData[i].Score;
        }
        return sum;
    }
    
    [Benchmark]
    public long SumScores_OptimalLayout()
    {
        long sum = 0;
        for (int i = 0; i < _optimalData.Length; i++)
        {
            if (_optimalData[i].IsAlive)
                sum += _optimalData[i].Score;
        }
        return sum;
    }
}

// Results (approximate):
// |                   Method |     Mean | Allocated |
// |------------------------- |---------:|----------:|
// | SumScores_PoorLayout     | 8.42 μs  |   400 KB  |
// | SumScores_OptimalLayout  | 5.67 μs  |   240 KB  |  (32% faster, 40% less memory)
```

## Bit Packing for Extreme Density

When every byte counts:

```csharp
[StructLayout(LayoutKind.Explicit, Size = 16)]
public struct CompactEntity
{
    [FieldOffset(0)]  public ulong EntityId;     // 8 bytes
    
    // Pack multiple values into 8 bytes
    [FieldOffset(8)]  public ushort Health;      // 2 bytes
    [FieldOffset(10)] public ushort Mana;        // 2 bytes
    [FieldOffset(12)] public byte Level;         // 1 byte
    [FieldOffset(13)] public byte Team;          // 1 byte
    [FieldOffset(14)] private ushort _flags;     // 2 bytes for bit flags
    
    // Flags as properties (no extra memory)
    public bool IsAlive
    {
        get => (_flags & 0b0001) != 0;
        set => _flags = value ? (ushort)(_flags | 0b0001) : (ushort)(_flags & ~0b0001);
    }
    
    public bool IsInvincible
    {
        get => (_flags & 0b0010) != 0;
        set => _flags = value ? (ushort)(_flags | 0b0010) : (ushort)(_flags & ~0b0010);
    }
    
    public bool HasPowerUp
    {
        get => (_flags & 0b0100) != 0;
        set => _flags = value ? (ushort)(_flags | 0b0100) : (ushort)(_flags & ~0b0100);
    }
    
    // 13 bits still available for more flags!
}

// Usage
var entity = new CompactEntity
{
    EntityId = 12345,
    Health = 100,
    Mana = 50,
    Level = 10,
    Team = 1,
    IsAlive = true,
    IsInvincible = false,
    HasPowerUp = true
};

Console.WriteLine(Unsafe.SizeOf<CompactEntity>());  // 16 bytes for all this data!
```

## Union Types for Space Efficiency

When fields are mutually exclusive:

```csharp
[StructLayout(LayoutKind.Explicit)]
public struct NetworkMessage
{
    [FieldOffset(0)] public MessageType Type;  // 4 bytes
    
    // Different message payloads share the same memory
    [FieldOffset(4)] public PlayerMoveData MoveData;
    [FieldOffset(4)] public ChatMessageData ChatData;
    [FieldOffset(4)] public PlayerDamageData DamageData;
    
    // Total size = 4 + max(sizeof(PlayerMoveData), sizeof(ChatMessageData), sizeof(PlayerDamageData))
}

public enum MessageType { Move, Chat, Damage }

public struct PlayerMoveData
{
    public float X;
    public float Y;
    public float Z;
}

public struct ChatMessageData
{
    public int SenderId;
    public int MessageId;
    // Actual message text stored separately
}

public struct PlayerDamageData
{
    public int AttackerId;
    public int VictimId;
    public int Damage;
}

// Usage
var message = new NetworkMessage
{
    Type = MessageType.Move,
    MoveData = new PlayerMoveData { X = 10, Y = 5, Z = 3 }
};

switch (message.Type)
{
    case MessageType.Move:
        ProcessMove(message.MoveData);
        break;
    case MessageType.Chat:
        ProcessChat(message.ChatData);
        break;
    case MessageType.Damage:
        ProcessDamage(message.DamageData);
        break;
}
```

## Struct Padding Calculator

```csharp
public static class StructAnalyzer
{
    public static void AnalyzeLayout<T>() where T : struct
    {
        var type = typeof(T);
        var fields = type.GetFields(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic)
            .OrderBy(f => Marshal.OffsetOf(type, f.Name).ToInt32())
            .ToList();
        
        Console.WriteLine($"Struct: {type.Name}");
        Console.WriteLine($"Size: {Unsafe.SizeOf<T>()} bytes\n");
        
        int offset = 0;
        int totalPadding = 0;
        
        foreach (var field in fields)
        {
            var fieldOffset = Marshal.OffsetOf(type, field.Name).ToInt32();
            var fieldSize = Marshal.SizeOf(field.FieldType);
            
            // Calculate padding before this field
            int padding = fieldOffset - offset;
            if (padding > 0)
            {
                Console.WriteLine($"  [padding: {padding} bytes]");
                totalPadding += padding;
            }
            
            Console.WriteLine($"  {field.Name} (offset {fieldOffset}): {fieldSize} bytes");
            offset = fieldOffset + fieldSize;
        }
        
        // Padding at the end
        int endPadding = Unsafe.SizeOf<T>() - offset;
        if (endPadding > 0)
        {
            Console.WriteLine($"  [padding: {endPadding} bytes]");
            totalPadding += endPadding;
        }
        
        Console.WriteLine($"\nTotal padding: {totalPadding} bytes");
        Console.WriteLine($"Efficiency: {(1.0 - (double)totalPadding / Unsafe.SizeOf<T>()) * 100:F1}%");
    }
}

// Usage
StructAnalyzer.AnalyzeLayout<PlayerState>();
```

## Why It's a Problem

1. **Memory waste**: Poor ordering creates unnecessary padding
2. **Cache misses**: Separated fields cause multiple cache line loads
3. **Lower throughput**: Fewer structs fit in cache = more memory bandwidth consumed
4. **Hidden cost**: 40% memory overhead is invisible in source code

## Symptoms

- Struct sizes larger than expected
- Cache miss rates high in profiler
- Memory bandwidth bottleneck
- Performance degrades with larger data sets
- Many bool fields scattered through struct

## Benefits

- **Reduced memory usage**: 30-50% size reduction typical
- **Better cache utilization**: More structs per cache line
- **Improved throughput**: Less memory bandwidth consumed
- **Explicit control**: Know exactly how much memory each struct uses

## Rules of Thumb

1. **Order by size**: Largest fields first, smallest last
2. **Group by access pattern**: Hot fields together
3. **Power-of-2 alignment**: 8-byte fields on 8-byte boundaries
4. **Measure**: Use `Unsafe.SizeOf<T>()` and `Marshal.OffsetOf()`

## See Also

- [Memory Safety (Span)](./memory-safety-span.md) — zero-allocation patterns
- [Value Semantics](./value-semantics.md) — when to use structs vs classes
- [Snapshot Immutability](./snapshot-immutability.md) — immutable structs
