# Snapshot Testing

> Capture and verify complex outputs—compare against approved baselines to detect unintended changes.

## Problem

Testing complex outputs like JSON, HTML, or large object graphs with assertions is tedious and brittle. Changes to output format require updating many assertions.

## Pattern

Capture the output once, approve it as the baseline, then automatically compare future outputs against the approved snapshot. Update snapshots when changes are intentional.

## Libraries

- **Verify**: Popular .NET snapshot testing library
- **Snapshooter**: Alternative with xUnit integration
- **ApprovalTests**: Older but mature option

## Example

### ❌ Before - Manual Assertions

```csharp
[Fact]
public void SerializeOrder_ProducesCorrectJson()
{
    var order = OrderMother.DefaultOrder();
    
    var json = JsonSerializer.Serialize(order);
    
    // ❌ Tedious and brittle
    Assert.Contains("\"orderId\":", json);
    Assert.Contains("\"customerId\":", json);
    Assert.Contains("\"total\":", json);
    Assert.Contains("100.00", json);
    // ... many more assertions
}
```

### ✅ After - Snapshot Testing

```csharp
[Fact]
public Task SerializeOrder_ProducesCorrectJson()
{
    // Arrange
    var order = OrderMother.DefaultOrder();

    // Act
    var json = JsonSerializer.Serialize(order, new JsonSerializerOptions 
    { 
        WriteIndented = true 
    });

    // Assert - Verify against snapshot
    return Verify(json);
}
```

## Using Verify

### Setup

```csharp
// Add to test project
// dotnet add package Verify.Xunit

// ModuleInitializer.cs - runs once per test assembly
public static class ModuleInitializer
{
    [ModuleInitializer]
    public static void Init()
    {
        VerifySourceGenerators.Enable();
    }
}
```

### Basic Usage

```csharp
using VerifyXunit;

[UsesVerify] // Required attribute on test class
public class OrderSerializationTests
{
    [Fact]
    public Task SerializeOrder_MatchesSnapshot()
    {
        // Arrange
        var order = new Order(
            OrderId.Parse("00000000-0000-0000-0000-000000000001"),
            CustomerId.Parse("00000000-0000-0000-0000-000000000002"),
            Money.USD(100m)
        );

        // Act
        var json = JsonSerializer.Serialize(order, new JsonSerializerOptions
        {
            WriteIndented = true
        });

        // Assert - Creates .verified.txt file on first run
        return Verify(json);
    }
}
```

### Testing Objects

```csharp
[Fact]
public Task CreateOrder_ProducesCorrectStructure()
{
    // Arrange
    var customerId = CustomerId.Parse("00000000-0000-0000-0000-000000000001");
    var items = new[]
    {
        OrderItem.Create(ProductId.Parse("00000000-0000-0000-0000-000000000010"), 2, Money.USD(10m)),
        OrderItem.Create(ProductId.Parse("00000000-0000-0000-0000-000000000020"), 1, Money.USD(25m)),
    };

    // Act
    var order = Order.Create(customerId, items);

    // Assert - Verify entire object graph
    return Verify(order);
}
```

## Scrubbing Dynamic Data

```csharp
[Fact]
public Task CreateOrder_ScrubbingDynamicData()
{
    // Arrange
    var order = OrderMother.DefaultOrder();

    // Assert - Scrub time-dependent or random data
    return Verify(order)
        .ScrubMember("CreatedAt")
        .ScrubMember("OrderId")
        .ScrubMember("CustomerId");
}

[Fact]
public Task CreateOrder_ScrubbingWithCustom()
{
    var order = OrderMother.DefaultOrder();

    return Verify(order)
        .AddScrubber(json => 
        {
            // Replace GUIDs with placeholder
            return Regex.Replace(json, 
                @"[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}", 
                "GUID");
        });
}
```

## Testing HTML Output

```csharp
[Fact]
public Task RenderOrderEmail_ProducesCorrectHtml()
{
    // Arrange
    var order = OrderMother.DefaultOrder();
    var renderer = new OrderEmailRenderer();

    // Act
    var html = renderer.RenderConfirmationEmail(order);

    // Assert - Verify HTML structure
    return Verify(html)
        .UseExtension("html");
}
```

## Testing Multiple Snapshots

```csharp
[Theory]
[InlineData(OrderStatus.Pending)]
[InlineData(OrderStatus.Confirmed)]
[InlineData(OrderStatus.Shipped)]
public Task RenderOrder_ForEachStatus_ProducesCorrectOutput(OrderStatus status)
{
    // Arrange
    var order = OrderMother.OrderInStatus(status);
    var renderer = new OrderRenderer();

    // Act
    var output = renderer.Render(order);

    // Assert - Creates separate snapshot for each status
    return Verify(output)
        .UseParameters(status);
}
```

## Testing API Responses

```csharp
[Fact]
public async Task GetOrder_ReturnsCorrectResponse()
{
    // Arrange
    var orderId = OrderId.Parse("00000000-0000-0000-0000-000000000001");
    var order = OrderMother.DefaultOrder();
    repository.Add(order);

    var client = new TestHttpClient();

    // Act
    var response = await client.GetAsync($"/api/orders/{orderId}");
    var content = await response.Content.ReadAsStringAsync();

    // Assert
    return Verify(new
    {
        StatusCode = (int)response.StatusCode,
        Headers = response.Headers.ToDictionary(h => h.Key, h => h.Value),
        Body = JsonDocument.Parse(content)
    });
}
```

## Testing Error Messages

```csharp
[Fact]
public Task ValidateOrder_WithErrors_ProducesCorrectMessages()
{
    // Arrange
    var invalidOrder = OrderBuilder.Default()
        .WithEmptyCart()
        .WithNegativeTotal()
        .Build();

    // Act
    var result = invalidOrder.Validate();

    // Assert - Verify all error messages
    return Verify(result.Errors);
}
```

## Updating Snapshots

When output intentionally changes:

1. Review the diff between old and new snapshot
2. If correct, delete the `.received.` file to approve
3. Or use Verify's diff tool to approve changes

```bash
# Review all pending snapshot updates
dotnet test

# The test will show a diff and wait for approval
# After reviewing, the .received. files become .verified.
```

## Ignoring Members

```csharp
// Global settings
VerifierSettings.IgnoreMember<Order>(o => o.CreatedAt);
VerifierSettings.IgnoreMember<Order>(o => o.UpdatedAt);

// Per test
[Fact]
public Task Test()
{
    var order = OrderMother.DefaultOrder();
    
    return Verify(order)
        .IgnoreMember("CreatedAt");
}
```

## Custom Comparers

```csharp
[Fact]
public Task CompareOrders_IgnoringTimestamps()
{
    var order1 = OrderMother.DefaultOrder();
    var order2 = OrderMother.DefaultOrder();

    return Verify(new { order1, order2 })
        .ModifySerialization(settings =>
        {
            settings.IgnoreMember<DateTime>(nameof(DateTime.Ticks));
        });
}
```

## Guidelines

1. **Use snapshots for complex outputs** (JSON, HTML, large objects)
2. **Scrub dynamic data** (timestamps, GUIDs, random values)
3. **Review snapshot changes** carefully before approving
4. **Keep snapshots readable** (use formatting, extensions)
5. **Use parameters** for multiple related tests
6. **Commit snapshots** to version control

## When to Use Snapshot Testing

**Good for:**
- API response formats
- Serialized data structures
- HTML/XML output
- Error message collections
- Complex object graphs
- Generated code

**Not good for:**
- Simple values (use assertions)
- Behavior verification (use mocks)
- Calculations (use assertions)
- Binary data (hard to review)

## Benefits

1. **Comprehensive**: Captures entire output automatically
2. **Maintainable**: Update snapshots instead of many assertions
3. **Discoverable**: Easy to see what changed
4. **Fast**: Write tests quickly
5. **Documentation**: Snapshots show expected output

## Pitfalls

1. **Blind approval**: Review changes before approving
2. **Noise**: Scrub dynamic data properly
3. **Too many snapshots**: Group related tests
4. **Large snapshots**: Hard to review in diffs
5. **Implementation coupling**: Avoid testing implementation details

## Example Workflow

```csharp
[UsesVerify]
public class OrderReportTests
{
    // First run: Creates OrderReportTests.GenerateMonthlyReport.verified.txt
    [Fact]
    public Task GenerateMonthlyReport_ProducesCorrectFormat()
    {
        // Arrange
        var orders = new[]
        {
            OrderMother.OrderWithTotal(Money.USD(100m)),
            OrderMother.OrderWithTotal(Money.USD(200m)),
            OrderMother.OrderWithTotal(Money.USD(150m)),
        };

        var generator = new MonthlyReportGenerator();

        // Act
        var report = generator.Generate(orders, 
            year: 2024, 
            month: 1);

        // Assert
        return Verify(report)
            .UseExtension("txt")
            .ScrubMember("GeneratedAt");
    }
}
```

## See Also

- [Testing Domain Invariants](./testing-domain-invariants.md) — validating business rules
- [Testing Value Objects](./testing-value-objects.md) — simpler equality testing
- [Integration Test Patterns](./integration-test-patterns.md) — testing full stack
- [Parameterized Tests](./parameterized-tests.md) — testing multiple scenarios
