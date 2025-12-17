# Testing Switch Expressions

> Test switch expressions and exhaustiveness—verify all cases are handled and return correct values.

## Problem

Switch expressions require exhaustive handling of all cases. Tests must verify every branch returns correct values.

## Example

```csharp
public enum OrderStatus { Pending, Processing, Shipped, Delivered, Cancelled }

public static string GetStatusMessage(OrderStatus status) => status switch
{
    OrderStatus.Pending => "Order is pending",
    OrderStatus.Processing => "Order is being processed",
    OrderStatus.Shipped => "Order has been shipped",
    OrderStatus.Delivered => "Order has been delivered",
    OrderStatus.Cancelled => "Order was cancelled",
    _ => throw new ArgumentOutOfRangeException(nameof(status))
};

[Theory]
[InlineData(OrderStatus.Pending, "Order is pending")]
[InlineData(OrderStatus.Processing, "Order is being processed")]
[InlineData(OrderStatus.Shipped, "Order has been shipped")]
[InlineData(OrderStatus.Delivered, "Order has been delivered")]
[InlineData(OrderStatus.Cancelled, "Order was cancelled")]
public void GetStatusMessage_AllStatuses_ReturnsCorrectMessage(
    OrderStatus status, string expected)
{
    var result = GetStatusMessage(status);
    
    result.Should().Be(expected);
}
```

## Testing Nested Switch Expressions

```csharp
public static int GetPriority(OrderStatus status, bool isExpress) => (status, isExpress) switch
{
    (OrderStatus.Pending, true) => 1,
    (OrderStatus.Pending, false) => 3,
    (OrderStatus.Processing, true) => 2,
    (OrderStatus.Processing, false) => 4,
    _ => 5
};

[Theory]
[InlineData(OrderStatus.Pending, true, 1)]
[InlineData(OrderStatus.Pending, false, 3)]
public void GetPriority_Combinations_ReturnsCorrectPriority(
    OrderStatus status, bool isExpress, int expected)
{
    var result = GetPriority(status, isExpress);
    
    result.Should().Be(expected);
}
```

## See Also

- [Testing Pattern Matching](./testing-pattern-matching.md)
- [Testing Boolean Logic](./testing-boolean-logic.md)
