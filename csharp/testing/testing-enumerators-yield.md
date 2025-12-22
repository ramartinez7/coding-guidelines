# Testing Custom Enumerators and Yield

> Test custom enumerators and yield return—verify iteration, lazy evaluation, and state management.

## Problem

Custom enumerators use yield return for lazy evaluation. Tests must verify correct iteration order, lazy evaluation, and proper cleanup.

## Example

```csharp
public class Range
{
    public static IEnumerable<int> Generate(int start, int count)
    {
        for (int i = 0; i < count; i++)
        {
            yield return start + i;
        }
    }
}

[Fact]
public void Generate_ProducesCorrectSequence()
{
    var result = Range.Generate(5, 3).ToList();
    
    result.Should().Equal(5, 6, 7);
}

[Fact]
public void Generate_IsLazy_DoesNotEvaluateUntilIterated()
{
    var evaluationCount = 0;
    
    IEnumerable<int> Enumerate()
    {
        for (int i = 0; i < 5; i++)
        {
            evaluationCount++;
            yield return i;
        }
    }
    
    var sequence = Enumerate();
    evaluationCount.Should().Be(0);
    
    sequence.Take(2).ToList();
    evaluationCount.Should().Be(2);
}
```

## Testing Iterator State

```csharp
[Fact]
public void Iterator_MultipleEnumerations_IndependentState()
{
    var source = Range.Generate(1, 3);
    
    var first = source.ToList();
    var second = source.ToList();
    
    first.Should().Equal(second);
}
```

## See Also

- [Testing LINQ Queries](./testing-linq-queries.md)
- [Testing Collection Assertions](./testing-collection-assertions.md)
