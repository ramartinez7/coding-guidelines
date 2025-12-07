# Indexed Types (Type-Level Indices)

> Array indexing with integers risks out-of-bounds errors—use typed indices to make invalid indices unrepresentable at compile time.

## Problem

Array and collection access using integer indices provides no guarantee that the index is valid. Runtime bounds checking catches errors late, often in production. Indexed types make index validity part of the type system.

> **Note:** Examples use `Option<T>` for representing potentially missing values. See [Nullability vs. Optionality](./nullability-optionality.md) or use libraries like [LanguageExt](https://github.com/louthy/language-ext).

## Example

### ❌ Before

```csharp
public class GameBoard
{
    private readonly string[,] _cells = new string[8, 8];
    
    public string GetCell(int row, int column)
    {
        // Runtime bounds checking only
        if (row < 0 || row >= 8)
            throw new ArgumentOutOfRangeException(nameof(row));
        
        if (column < 0 || column >= 8)
            throw new ArgumentOutOfRangeException(nameof(column));
        
        return _cells[row, column];
    }
    
    public void SetCell(int row, int column, string value)
    {
        if (row < 0 || row >= 8)
            throw new ArgumentOutOfRangeException(nameof(row));
        
        if (column < 0 || column >= 8)
            throw new ArgumentOutOfRangeException(nameof(column));
        
        _cells[row, column] = value;
    }
}

// Problems:
// 1. Can pass invalid indices (compiles, fails at runtime)
// 2. Bounds checks repeated in every method
// 3. No compile-time guarantee
// 4. Easy to swap row/column parameters

var board = new GameBoard();
board.SetCell(10, 5, "X");  // Compiles, throws at runtime
board.SetCell(3, -1, "O");  // Compiles, throws at runtime
board.GetCell(5, 10);  // Compiles, throws at runtime
```

### ✅ After (With Indexed Types)

```csharp
/// <summary>
/// Index that is guaranteed to be within bounds for size N.
/// </summary>
public readonly record struct Index<TSize>(int Value)
    where TSize : ISize
{
    private Index(int value) : this()
    {
        Value = value;
    }
    
    /// <summary>
    /// Create index if within bounds, otherwise None.
    /// </summary>
    public static Option<Index<TSize>> Create(int value)
    {
        var size = TSize.Value;
        
        if (value < 0 || value >= size)
            return Option<Index<TSize>>.None;
        
        return Option<Index<TSize>>.Some(new Index<TSize>(value));
    }
    
    /// <summary>
    /// Create index, throwing if out of bounds.
    /// Use only when bounds are known at compile time.
    /// </summary>
    public static Index<TSize> Unsafe(int value)
    {
        var size = TSize.Value;
        
        if (value < 0 || value >= size)
            throw new ArgumentOutOfRangeException(nameof(value), 
                $"Index {value} is out of range [0, {size})");
        
        return new Index<TSize>(value);
    }
    
    public override string ToString() => $"Index({Value})";
}

/// <summary>
/// Marker interface for compile-time size constraints.
/// </summary>
public interface ISize
{
    static abstract int Value { get; }
}

/// <summary>
/// Size marker for 8x8 boards (chess, checkers).
/// </summary>
public struct Size8 : ISize
{
    public static int Value => 8;
}

/// <summary>
/// Size marker for 3x3 grids (tic-tac-toe).
/// </summary>
public struct Size3 : ISize
{
    public static int Value => 3;
}

/// <summary>
/// Type-safe 2D array with compile-time size.
/// </summary>
public sealed class Grid<TSize, T> where TSize : ISize
{
    private readonly T[,] _cells;
    private readonly int _size;
    
    public Grid()
    {
        _size = TSize.Value;
        _cells = new T[_size, _size];
    }
    
    /// <summary>
    /// Access cell—no bounds checking needed!
    /// Type guarantees index is valid.
    /// </summary>
    public T Get(Index<TSize> row, Index<TSize> column)
    {
        // No bounds check needed—types guarantee validity
        return _cells[row.Value, column.Value];
    }
    
    public void Set(Index<TSize> row, Index<TSize> column, T value)
    {
        // No bounds check needed
        _cells[row.Value, column.Value] = value;
    }
    
    /// <summary>
    /// Get all valid indices for this grid size.
    /// </summary>
    public IEnumerable<Index<TSize>> AllIndices()
    {
        for (int i = 0; i < _size; i++)
        {
            yield return Index<TSize>.Unsafe(i);
        }
    }
}

// Usage: 8x8 chess board
public class ChessBoard
{
    private readonly Grid<Size8, string> _board = new();
    
    // Type signature proves indices are valid
    public string GetPiece(Index<Size8> row, Index<Size8> column)
    {
        // No bounds checking—type guarantees validity
        return _board.Get(row, column);
    }
    
    public void SetPiece(Index<Size8> row, Index<Size8> column, string piece)
    {
        _board.Set(row, column, piece);
    }
    
    public void InitializeBoard()
    {
        // Compile-time safe initialization
        var row0 = Index<Size8>.Unsafe(0);
        var col0 = Index<Size8>.Unsafe(0);
        
        SetPiece(row0, col0, "Rook");
        
        // Iterate all positions safely
        foreach (var row in _board.AllIndices())
        {
            foreach (var col in _board.AllIndices())
            {
                SetPiece(row, col, "Empty");
            }
        }
    }
}

// Usage: Create indices safely at boundaries
public class ChessGame
{
    ChessBoard board = new();
    
    void MovePiece(int fromRow, int fromCol, int toRow, int toCol)
    {
        // Parse user input into valid indices
        var from = (
            Row: Index<Size8>.Create(fromRow),
            Col: Index<Size8>.Create(fromCol)
        );
        
        var to = (
            Row: Index<Size8>.Create(toRow),
            Col: Index<Size8>.Create(toCol)
        );
        
        // Pattern match on results
        if (from.Row.IsSome && from.Col.IsSome && to.Row.IsSome && to.Col.IsSome)
        {
            var piece = board.GetPiece(from.Row.Value, from.Col.Value);
            board.SetPiece(to.Row.Value, to.Col.Value, piece);
            board.SetPiece(from.Row.Value, from.Col.Value, "Empty");
        }
        else
        {
            throw new ArgumentException("Invalid move coordinates");
        }
    }
}

// Cannot create invalid indices
// var invalid = new Index<Size8>(-1);  // ❌ Won't compile: private constructor
// var invalid2 = Index<Size8>.Create(10);  // Returns Option.None

// Type system prevents row/column swaps
// board.SetPiece(col, row, piece);  // ❌ Won't compile if col/row are typed differently
```

## Matrix with Typed Dimensions

```csharp
/// <summary>
/// Matrix with compile-time row and column dimensions.
/// </summary>
public sealed class Matrix<TRows, TCols, T> 
    where TRows : ISize 
    where TCols : ISize
{
    private readonly T[,] _data;
    
    public Matrix()
    {
        _data = new T[TRows.Value, TCols.Value];
    }
    
    public T Get(Index<TRows> row, Index<TCols> column)
    {
        return _data[row.Value, column.Value];
    }
    
    public void Set(Index<TRows> row, Index<TCols> column, T value)
    {
        _data[row.Value, column.Value] = value;
    }
    
    /// <summary>
    /// Matrix multiplication with compile-time dimension checking.
    /// Can only multiply if this.Cols == other.Rows.
    /// </summary>
    public Matrix<TRows, TOtherCols, T> Multiply<TOtherCols>(
        Matrix<TCols, TOtherCols, T> other)
        where TOtherCols : ISize
    {
        // Type system ensures dimensions match!
        // Cannot compile if dimensions are incompatible.
        
        var result = new Matrix<TRows, TOtherCols, T>();
        
        // Multiplication logic here...
        
        return result;
    }
}

// Define common sizes
public struct Size2 : ISize { public static int Value => 2; }
public struct Size4 : ISize { public static int Value => 4; }

// Usage
var matrix2x4 = new Matrix<Size2, Size4, double>();
var matrix4x2 = new Matrix<Size4, Size2, double>();

// Type-safe multiplication: (2x4) * (4x2) = (2x2)
var result = matrix2x4.Multiply(matrix4x2);  // ✅ Returns Matrix<Size2, Size2, double>

// Won't compile: dimension mismatch
// var invalid = matrix2x4.Multiply(matrix2x4);  // ❌ Can't multiply (2x4) * (2x4)
```

## Fixed-Size Arrays

```csharp
/// <summary>
/// Array with compile-time size guarantee.
/// </summary>
public sealed class FixedArray<TSize, T> where TSize : ISize
{
    private readonly T[] _items;
    
    public FixedArray()
    {
        _items = new T[TSize.Value];
    }
    
    public FixedArray(params T[] items)
    {
        if (items.Length != TSize.Value)
            throw new ArgumentException(
                $"Expected {TSize.Value} items, got {items.Length}");
        
        _items = items;
    }
    
    public T Get(Index<TSize> index) => _items[index.Value];
    
    public void Set(Index<TSize> index, T value) => _items[index.Value] = value;
    
    public int Length => TSize.Value;
    
    // Safe enumeration
    public IEnumerator<T> GetEnumerator()
    {
        foreach (var item in _items)
        {
            yield return item;
        }
    }
}

// RGB color as fixed 3-element array
public class RgbColor
{
    private readonly FixedArray<Size3, byte> _channels = new();
    
    public byte Red
    {
        get => _channels.Get(Index<Size3>.Unsafe(0));
        set => _channels.Set(Index<Size3>.Unsafe(0), value);
    }
    
    public byte Green
    {
        get => _channels.Get(Index<Size3>.Unsafe(1));
        set => _channels.Set(Index<Size3>.Unsafe(1), value);
    }
    
    public byte Blue
    {
        get => _channels.Get(Index<Size3>.Unsafe(2));
        set => _channels.Set(Index<Size3>.Unsafe(2), value);
    }
}
```

## Range-Constrained Indices

```csharp
/// <summary>
/// Index constrained to a specific range.
/// </summary>
public readonly record struct RangeIndex<TMin, TMax>(int Value)
    where TMin : ISize
    where TMax : ISize
{
    private RangeIndex(int value) : this()
    {
        Value = value;
    }
    
    public static Option<RangeIndex<TMin, TMax>> Create(int value)
    {
        var min = TMin.Value;
        var max = TMax.Value;
        
        if (value < min || value > max)
            return Option<RangeIndex<TMin, TMax>>.None;
        
        return Option<RangeIndex<TMin, TMax>>.Some(
            new RangeIndex<TMin, TMax>(value));
    }
}

// Week day index: 0-6
public struct Zero : ISize { public static int Value => 0; }
public struct Six : ISize { public static int Value => 6; }

public using WeekDayIndex = RangeIndex<Zero, Six>;

public class Calendar
{
    string GetDayName(WeekDayIndex dayIndex) => dayIndex.Value switch
    {
        0 => "Sunday",
        1 => "Monday",
        2 => "Tuesday",
        3 => "Wednesday",
        4 => "Thursday",
        5 => "Friday",
        6 => "Saturday",
        _ => throw new UnreachableException()
    };
}
```

## Why It's a Problem

1. **Runtime bounds errors**: Index validation happens too late
2. **Repeated checks**: Every array access needs validation
3. **No compile-time guarantee**: Easy to use wrong index
4. **Performance**: Runtime bounds checking overhead

## Symptoms

- `IndexOutOfRangeException` in production
- Bounds checks at start of every method
- Unit tests specifically for index validation
- Comments like "// Ensure index is valid"

## Benefits

- **Compile-time bounds checking**: Invalid indices don't compile
- **No runtime overhead**: Bounds checked once at creation
- **Self-documenting**: Type shows valid range
- **Type safety**: Can't swap indices of different sizes

## Trade-offs

- **Complexity**: More types to manage
- **Verbosity**: Must create indices explicitly
- **Generic constraints**: Can be complex with static abstract members
- **Learning curve**: Team must understand indexed types

## See Also

- [Refinement Types](./refinement-types.md) — structural constraints on types
- [Phantom Types](./phantom-types.md) — compile-time state tracking
- [Strongly Typed IDs](./strongly-typed-ids.md) — type safety for identifiers
- [Units of Measure](./units-of-measure.md) — dimensional analysis
