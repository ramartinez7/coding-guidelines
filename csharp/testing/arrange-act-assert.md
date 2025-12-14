# Arrange-Act-Assert (AAA) Pattern

> Structure tests in three distinct phases—setup, execution, and verification—for clarity and maintainability.

## Problem

Tests that mix setup, execution, and verification become hard to understand and maintain. Without clear structure, readers struggle to identify what is being tested and why failures occur.

## Example

### ❌ Before

```csharp
[Fact]
public void ProcessOrder()
{
    var repository = new InMemoryOrderRepository();
    var order = new Order(OrderId.New(), CustomerId.New(), Money.USD(100m));
    repository.Add(order);
    var service = new OrderService(repository);
    var processor = new OrderProcessor(service);
    var result = processor.Process(order.Id);
    Assert.True(result.IsSuccess);
    Assert.Equal(OrderStatus.Processed, order.Status);
    var retrieved = repository.Get(order.Id);
    Assert.NotNull(retrieved);
}
```

### ✅ After

```csharp
[Fact]
public void ProcessOrder_ValidOrder_MarksOrderAsProcessed()
{
    // Arrange
    var customerId = CustomerId.New();
    var orderId = OrderId.New();
    var order = Order.Create(orderId, customerId, Money.USD(100m));
    
    var repository = new InMemoryOrderRepository();
    repository.Add(order);
    
    var service = new OrderService(repository);
    var processor = new OrderProcessor(service);

    // Act
    var result = processor.Process(orderId);

    // Assert
    Assert.True(result.IsSuccess);
    
    var processedOrder = repository.Get(orderId);
    Assert.Equal(OrderStatus.Processed, processedOrder.Status);
}
```

## Why It's Important

1. **Readability**: Clear structure makes test intent obvious
2. **Debugging**: Quickly identify which phase failed
3. **Maintenance**: Easy to modify setup or assertions independently
4. **Consistency**: Team follows same pattern across all tests

## Guidelines

**Arrange Phase**
- Set up test dependencies and input data
- Create mocks, stubs, or fakes
- Configure system under test
- Should be the longest section for complex scenarios

**Act Phase**
- Execute the operation being tested
- Usually a single method call
- Should be the shortest section (often one line)

**Assert Phase**
- Verify the expected outcome
- Check return values, state changes, or interactions
- Can have multiple assertions if verifying same logical concept

## Additional Tips

- Use blank lines to separate the three phases
- Add comment markers (`// Arrange`, `// Act`, `// Assert`) for clarity
- Keep Act phase minimal—if it's complex, you may be testing too much
- If Arrange is very long, consider using test data builders

## See Also

- [AAA Pattern with Clear Boundaries](./aaa-clear-boundaries.md) — visual separation techniques
- [Test Data Builders](./test-data-builders.md) — simplify complex Arrange phases
- [One Assertion Per Test](./one-assertion-per-test.md) — keep Assert focused
- [Test Naming Conventions](./test-naming-conventions.md) — communicate test intent
