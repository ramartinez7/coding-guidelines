# Testing Pattern Matching

> Test pattern matching expressions—verify type patterns, property patterns, and exhaustiveness.

## Problem

Pattern matching enables concise type checking and deconstruction. Tests must verify all patterns are handled correctly.

## Example

```csharp
public record Shape;
public record Circle(double Radius) : Shape;
public record Rectangle(double Width, double Height) : Shape;

public static class ShapeCalculator
{
    public static double CalculateArea(Shape shape) => shape switch
    {
        Circle c => Math.PI * c.Radius * c.Radius,
        Rectangle r => r.Width * r.Height,
        _ => throw new ArgumentException("Unknown shape")
    };
}

[Fact]
public void CalculateArea_Circle_ReturnsCorrectArea()
{
    var circle = new Circle(5);
    
    var area = ShapeCalculator.CalculateArea(circle);
    
    area.Should().BeApproximately(78.54, 0.01);
}

[Fact]
public void CalculateArea_Rectangle_ReturnsCorrectArea()
{
    var rectangle = new Rectangle(4, 6);
    
    var area = ShapeCalculator.CalculateArea(rectangle);
    
    area.Should().Be(24);
}
```

## Testing Property Patterns

```csharp
public static string Describe(Shape shape) => shape switch
{
    Circle { Radius: > 10 } => "Large circle",
    Circle => "Small circle",
    Rectangle { Width: var w, Height: var h } when w == h => "Square",
    Rectangle => "Rectangle",
    _ => "Unknown"
};

[Theory]
[InlineData(15, "Large circle")]
[InlineData(5, "Small circle")]
public void Describe_Circle_UsesRadiusPattern(double radius, string expected)
{
    var circle = new Circle(radius);
    
    var result = ShapeCalculator.Describe(circle);
    
    result.Should().Be(expected);
}
```

## See Also

- [Testing Switch Expressions](./testing-switch-expressions.md)
- [Testing Discriminated Unions](./testing-discriminated-unions.md)
