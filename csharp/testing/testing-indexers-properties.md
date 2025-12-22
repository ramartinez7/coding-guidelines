# Testing Indexers and Computed Properties

> Test indexers and computed properties—verify getter/setter behavior, bounds checking, and calculations.

## Problem

Indexers and computed properties encapsulate logic. Tests must verify they calculate correctly and handle invalid indices.

## Example

```csharp
public class Matrix
{
    private readonly int[,] data;
    
    public int this[int row, int col]
    {
        get
        {
            if (row < 0 || row >= Rows || col < 0 || col >= Cols)
                throw new IndexOutOfRangeException();
            return data[row, col];
        }
        set
        {
            if (row < 0 || row >= Rows || col < 0 || col >= Cols)
                throw new IndexOutOfRangeException();
            data[row, col] = value;
        }
    }
}

[Fact]
public void Indexer_ValidIndices_ReturnsValue()
{
    var matrix = new Matrix(3, 3);
    matrix[1, 1] = 42;
    
    matrix[1, 1].Should().Be(42);
}

[Fact]
public void Indexer_InvalidRow_Throws()
{
    var matrix = new Matrix(3, 3);
    
    Action act = () => { var _ = matrix[10, 0]; };
    
    act.Should().Throw<IndexOutOfRangeException>();
}
```

## Testing Computed Properties

```csharp
public record Rectangle(double Width, double Height)
{
    public double Area => Width * Height;
    public double Perimeter => 2 * (Width + Height);
}

[Fact]
public void Area_ComputedFromDimensions()
{
    var rect = new Rectangle(5, 10);
    
    rect.Area.Should().Be(50);
}
```

## See Also

- [Testing Guard Clauses](./testing-guard-clauses.md)
- [Testing Exceptions](./testing-exceptions.md)
