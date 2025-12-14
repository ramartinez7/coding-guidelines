# AAA Pattern with Clear Boundaries

> Use whitespace and comments to visually separate test phases—make the structure obvious at a glance.

## Problem

Even when following the Arrange-Act-Assert pattern, tests without clear visual boundaries become a wall of code that's hard to scan and understand quickly.

## Example

### ❌ Before - No Clear Boundaries

```csharp
[Fact]
public void ProcessOrder_WithValidOrder_UpdatesStatusAndSendsNotification()
{
    var customerId = CustomerId.New();
    var orderId = OrderId.New();
    var order = Order.Create(orderId, customerId, Money.USD(100m));
    var repository = new InMemoryOrderRepository();
    repository.Add(order);
    var emailService = new MockEmailService();
    var processor = new OrderProcessor(repository, emailService);
    var result = processor.Process(orderId);
    Assert.True(result.IsSuccess);
    var processed = repository.Get(orderId);
    Assert.Equal(OrderStatus.Processed, processed.Status);
    Assert.Single(emailService.SentEmails);
    Assert.Equal(customerId.ToString(), emailService.SentEmails[0].RecipientId);
}
```

### ✅ After - Clear Visual Separation

```csharp
[Fact]
public void ProcessOrder_WithValidOrder_UpdatesStatusAndSendsNotification()
{
    // Arrange
    var customerId = CustomerId.New();
    var orderId = OrderId.New();
    var order = Order.Create(orderId, customerId, Money.USD(100m));
    
    var repository = new InMemoryOrderRepository();
    repository.Add(order);
    
    var emailService = new MockEmailService();
    var processor = new OrderProcessor(repository, emailService);

    // Act
    var result = processor.Process(orderId);

    // Assert
    Assert.True(result.IsSuccess);
    
    var processed = repository.Get(orderId);
    Assert.Equal(OrderStatus.Processed, processed.Status);
    
    Assert.Single(emailService.SentEmails);
    Assert.Equal(customerId.ToString(), emailService.SentEmails[0].RecipientId);
}
```

## Visual Separation Techniques

### 1. Blank Lines Between Phases

Always use blank lines to separate Arrange, Act, and Assert.

```csharp
[Fact]
public void Test()
{
    // Arrange
    var input = "test";

    // Act
    var result = Process(input);

    // Assert
    Assert.NotNull(result);
}
```

### 2. Comment Markers

Use `// Arrange`, `// Act`, `// Assert` comments consistently.

```csharp
// ✅ Always include all three markers
[Fact]
public void Test()
{
    // Arrange
    var data = CreateTestData();

    // Act
    var result = Process(data);

    // Assert
    Assert.True(result.IsSuccess);
}
```

### 3. Blank Lines Within Phases

Group related setup within the Arrange phase.

```csharp
[Fact]
public void ComplexTest()
{
    // Arrange
    var customerId = CustomerId.New();
    var productId = ProductId.New();
    var quantity = 5;
    
    var product = Product.Create(productId, Money.USD(10m));
    var catalog = new InMemoryCatalog();
    catalog.Add(product);
    
    var customer = Customer.Create(customerId, "John Doe");
    var customerRepository = new InMemoryCustomerRepository();
    customerRepository.Add(customer);
    
    var orderService = new OrderService(catalog, customerRepository);

    // Act
    var result = orderService.CreateOrder(customerId, productId, quantity);

    // Assert
    Assert.True(result.IsSuccess);
    Assert.Equal(Money.USD(50m), result.Value.Total);
}
```

### 4. Helper Methods for Complex Setup

When Arrange grows too large, extract to helper methods.

```csharp
[Fact]
public void ProcessOrder_WithComplexSetup_Succeeds()
{
    // Arrange
    var (order, processor) = CreateOrderAndProcessor();

    // Act
    var result = processor.Process(order.Id);

    // Assert
    Assert.True(result.IsSuccess);
}

private (Order, OrderProcessor) CreateOrderAndProcessor()
{
    var customer = Customer.Create(CustomerId.New(), "John Doe");
    var product = Product.Create(ProductId.New(), Money.USD(10m));
    var order = Order.Create(customer.Id, product.Id, quantity: 1);
    
    var repository = new InMemoryOrderRepository();
    repository.Add(order);
    
    var processor = new OrderProcessor(repository);
    
    return (order, processor);
}
```

## Patterns for Assert Phase

### Group Related Assertions

```csharp
// Assert
// Verify success
Assert.True(result.IsSuccess);
Assert.NotNull(result.Value);

// Verify order state
Assert.Equal(OrderStatus.Processed, result.Value.Status);
Assert.Equal(expectedTotal, result.Value.Total);

// Verify side effects
Assert.Single(emailService.SentEmails);
Assert.Equal(customer.Email, emailService.SentEmails[0].Recipient);
```

### Use Comments to Explain Complex Assertions

```csharp
// Assert
// The order should be marked as processed
var retrieved = repository.Get(orderId);
Assert.Equal(OrderStatus.Processed, retrieved.Status);

// An audit log entry should be created
var logs = auditService.GetLogs(orderId);
Assert.Single(logs);
Assert.Equal(AuditAction.OrderProcessed, logs[0].Action);
```

## Complete Example

```csharp
[Fact]
public void CreateOrder_WithMultipleItems_CalculatesCorrectTotal()
{
    // Arrange
    // Setup products
    var product1 = Product.Create(ProductId.New(), "Widget", Money.USD(10m));
    var product2 = Product.Create(ProductId.New(), "Gadget", Money.USD(25m));
    
    var catalog = new InMemoryCatalog();
    catalog.Add(product1);
    catalog.Add(product2);
    
    // Setup customer
    var customerId = CustomerId.New();
    var customer = Customer.Create(customerId, "Jane Smith");
    var customerRepo = new InMemoryCustomerRepository();
    customerRepo.Add(customer);
    
    // Setup order service
    var orderService = new OrderService(catalog, customerRepo);
    
    // Define order items
    var items = new[]
    {
        new OrderItem(product1.Id, quantity: 2),  // 2 * $10 = $20
        new OrderItem(product2.Id, quantity: 3),  // 3 * $25 = $75
    };

    // Act
    var result = orderService.CreateOrder(customerId, items);

    // Assert
    // Verify order creation succeeded
    Assert.True(result.IsSuccess);
    Assert.NotNull(result.Value);
    
    // Verify order totals
    var order = result.Value;
    Assert.Equal(Money.USD(95m), order.Total);  // $20 + $75
    Assert.Equal(2, order.LineItems.Count);
    
    // Verify line items
    var widgetLine = order.LineItems.First(x => x.ProductId == product1.Id);
    Assert.Equal(2, widgetLine.Quantity);
    Assert.Equal(Money.USD(20m), widgetLine.Subtotal);
    
    var gadgetLine = order.LineItems.First(x => x.ProductId == product2.Id);
    Assert.Equal(3, gadgetLine.Quantity);
    Assert.Equal(Money.USD(75m), gadgetLine.Subtotal);
}
```

## Benefits

1. **Scanability**: Quickly identify test structure
2. **Debugging**: Immediately see which phase failed
3. **Maintenance**: Easy to add/modify setup or assertions
4. **Code reviews**: Reviewers can quickly understand test intent
5. **Consistency**: All tests follow same visual pattern

## Guidelines

- **Always** use blank lines between phases
- **Always** include `// Arrange`, `// Act`, `// Assert` comments
- Use blank lines within phases to group related setup
- Add explanatory comments for complex assertions
- Keep Act phase minimal (usually one line)
- Extract complex setup to helper methods

## See Also

- [Arrange-Act-Assert Pattern](./arrange-act-assert.md) — AAA fundamentals
- [Test Naming Conventions](./test-naming-conventions.md) — making tests self-documenting
- [Test Data Builders](./test-data-builders.md) — simplifying complex Arrange phases
- [One Assertion Per Test](./one-assertion-per-test.md) — keeping Assert focused
