# Dependent Types Emulation (Types That Depend on Values)

> Types that can't express relationships between values lead to runtime checks—emulate dependent types to encode value relationships in the type system.

## Problem

C# doesn't support true dependent types, but we can emulate them to encode relationships between values.

## Example

### ✅ Array with Length in Type

```csharp
// Emulate: Array<T, N> where N is the length
public sealed record FixedArray<T>
{
    private readonly T[] _items;
    public int Length { get; }
    
    private FixedArray(T[] items, int length)
    {
        _items = items;
        Length = length;
    }
    
    public static Result<FixedArray<T>, string> Create(T[] items, int expectedLength)
    {
        if (items.Length != expectedLength)
            return Result<FixedArray<T>, string>.Failure(
                $"Expected {expectedLength} items, got {items.Length}");
        
        return Result<FixedArray<T>, string>.Success(
            new FixedArray<T>(items, expectedLength));
    }
    
    public T this[int index]
    {
        get
        {
            if (index < 0 || index >= Length)
                throw new IndexOutOfRangeException();
            return _items[index];
        }
    }
}

// Usage: Type encodes expected length
var arrayResult = FixedArray<int>.Create(new[] { 1, 2, 3 }, expectedLength: 3);
```

### ✅ Bounded Integer

```csharp
// Emulate: Int<Min, Max>
public readonly record struct BoundedInt
{
    public int Value { get; }
    public int Min { get; }
    public int Max { get; }
    
    private BoundedInt(int value, int min, int max)
    {
        Value = value;
        Min = min;
        Max = max;
    }
    
    public static Result<BoundedInt, string> Create(int value, int min, int max)
    {
        if (value < min || value > max)
            return Result<BoundedInt, string>.Failure(
                $"Value {value} must be between {min} and {max}");
        
        return Result<BoundedInt, string>.Success(new BoundedInt(value, min, max));
    }
}

// Domain-specific bounded types
public sealed record Age
{
    private readonly BoundedInt _value;
    
    private Age(BoundedInt value) => _value = value;
    
    public static Result<Age, string> Create(int age)
    {
        var result = BoundedInt.Create(age, min: 0, max: 150);
        return result.IsSuccess
            ? Result<Age, string>.Success(new Age(result.Value))
            : Result<Age, string>.Failure(result.Error);
    }
}
```

## See Also

- [Phantom Types](./phantom-types.md)
- [Type Witnesses](./type-witnesses.md)
- [Indexed Types](./indexed-types.md)
