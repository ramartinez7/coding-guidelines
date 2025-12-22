# Unit Testing Best Practices

> Write clear, maintainable, and reliable unit tests that serve as documentation and catch regressions early.

## Problem

Poor tests are brittle, hard to understand, test implementation instead of behavior, and provide false confidence.

## Example

### ❌ Before

```csharp
[Test]
public void Test1()
{
    var service = new OrderService(new OrderRepository(), new PaymentService());
    var order = new Order();
    order.Items.Add(new OrderItem { Price = 10, Quantity = 2 });
    var result = service.Process(order);
    Assert.IsTrue(result);
}
```

### ✅ After

```csharp
[Test]
public void ProcessOrder_WithValidOrder_CalculatesTotalCorrectly()
{
    // Arrange
    var order = new OrderBuilder()
        .WithItem(price: 10m, quantity: 2)
        .Build();

    var sut = new OrderService(
        new FakeOrderRepository(),
        new FakePaymentService());

    // Act
    var result = sut.ProcessOrder(order);

    // Assert
    Assert.That(result.IsSuccess, Is.True);
    Assert.That(result.Value.Total, Is.EqualTo(20m));
}
```

## Best Practices

### 1. Use AAA Pattern (Arrange-Act-Assert)

```csharp
[Test]
public void CalculateDiscount_ForPremiumCustomer_Returns20Percent()
{
    // Arrange
    var customer = new Customer { IsPremium = true };
    var order = new Order { Total = 100m, Customer = customer };
    var calculator = new DiscountCalculator();

    // Act
    var discount = calculator.CalculateDiscount(order);

    // Assert
    Assert.That(discount, Is.EqualTo(20m));
}
```

### 2. One Assert Per Test (Usually)

```csharp
// ❌ Multiple unrelated assertions
[Test]
public void TestOrder()
{
    var order = CreateOrder();
    Assert.That(order.Id, Is.Not.Null);
    Assert.That(order.Total, Is.GreaterThan(0));
    Assert.That(order.Status, Is.EqualTo(OrderStatus.Pending));
}

// ✅ Focused single assertion
[Test]
public void NewOrder_HasPendingStatus()
{
    var order = CreateOrder();
    Assert.That(order.Status, Is.EqualTo(OrderStatus.Pending));
}

[Test]
public void NewOrder_HasPositiveTotal()
{
    var order = CreateOrder();
    Assert.That(order.Total, Is.GreaterThan(0));
}

// ✅ Multiple assertions OK when testing one logical concept
[Test]
public void OrderTotal_CalculatesCorrectly()
{
    var order = new OrderBuilder()
        .WithItem(price: 10m, quantity: 2)
        .WithItem(price: 5m, quantity: 3)
        .Build();

    Assert.Multiple(() =>
    {
        Assert.That(order.Subtotal, Is.EqualTo(35m));
        Assert.That(order.Tax, Is.EqualTo(2.80m));
        Assert.That(order.Total, Is.EqualTo(37.80m));
    });
}
```

### 3. Use Descriptive Test Names

```csharp
// ❌ Unclear names
[Test]
public void Test1() { }

[Test]
public void TestOrder() { }

// ✅ Clear, descriptive names
[Test]
public void CalculateTotal_WithMultipleItems_SumsAllPrices() { }

[Test]
public void ProcessOrder_WhenPaymentFails_ReturnsError() { }

[Test]
public void CreateUser_WithDuplicateEmail_ThrowsException() { }
```

### 4. Use Test Builders for Complex Objects

```csharp
// ✅ Test builder pattern
public class OrderBuilder
{
    private List<OrderItem> items = new();
    private Customer customer = new Customer { Name = "Test" };
    private OrderStatus status = OrderStatus.Pending;

    public OrderBuilder WithItem(decimal price, int quantity)
    {
        items.Add(new OrderItem { Price = price, Quantity = quantity });
        return this;
    }

    public OrderBuilder WithCustomer(Customer customer)
    {
        this.customer = customer;
        return this;
    }

    public OrderBuilder WithStatus(OrderStatus status)
    {
        this.status = status;
        return this;
    }

    public Order Build()
    {
        return new Order
        {
            Items = items,
            Customer = customer,
            Status = status
        };
    }
}

// Usage
var order = new OrderBuilder()
    .WithItem(price: 10m, quantity: 2)
    .WithCustomer(new Customer { IsPremium = true })
    .Build();
```

### 5. Test Behavior, Not Implementation

```csharp
// ❌ Testing implementation details
[Test]
public void ProcessOrder_CallsRepositorySaveMethod()
{
    var mockRepo = new Mock<IOrderRepository>();
    var service = new OrderService(mockRepo.Object);

    service.ProcessOrder(order);

    mockRepo.Verify(r => r.Save(It.IsAny<Order>()), Times.Once);
}

// ✅ Testing behavior/outcome
[Test]
public void ProcessOrder_ValidOrder_OrderIsPersisted()
{
    var repository = new FakeOrderRepository();
    var service = new OrderService(repository);

    service.ProcessOrder(order);

    Assert.That(repository.SavedOrders, Has.Member(order));
}
```

### 6. Use Meaningful Fake/Stub Implementations

```csharp
// ✅ Simple fake for testing
public class FakeOrderRepository : IOrderRepository
{
    public List<Order> SavedOrders { get; } = new();
    private readonly Dictionary<OrderId, Order> orders = new();

    public void Save(Order order)
    {
        orders[order.Id] = order;
        SavedOrders.Add(order);
    }

    public Order? Find(OrderId id)
    {
        return orders.TryGetValue(id, out var order) ? order : null;
    }
}
```

### 7. Test Edge Cases and Boundaries

```csharp
[TestCase(0)]
[TestCase(-1)]
[TestCase(int.MinValue)]
public void CalculateDiscount_WithZeroOrNegativeAmount_ReturnsZero(decimal amount)
{
    var result = DiscountCalculator.Calculate(amount);
    Assert.That(result, Is.EqualTo(0));
}

[TestCase(99.99)]
[TestCase(100.00)]
[TestCase(100.01)]
public void GetShippingCost_AroundFreeShippingThreshold_CalculatesCorrectly(
    decimal orderTotal)
{
    var cost = ShippingCalculator.Calculate(orderTotal);

    if (orderTotal >= 100m)
    {
        Assert.That(cost, Is.EqualTo(0));
    }
    else
    {
        Assert.That(cost, Is.GreaterThan(0));
    }
}
```

### 8. Keep Tests Independent

```csharp
// ❌ Tests depend on execution order
private static Order? sharedOrder;

[Test]
public void Test1_CreateOrder()
{
    sharedOrder = CreateOrder();
}

[Test]
public void Test2_ProcessOrder()
{
    ProcessOrder(sharedOrder);  // Depends on Test1!
}

// ✅ Independent tests
[Test]
public void CreateOrder_ReturnsValidOrder()
{
    var order = CreateOrder();
    Assert.That(order, Is.Not.Null);
}

[Test]
public void ProcessOrder_WithValidOrder_Succeeds()
{
    var order = CreateOrder();  // Create fresh state
    var result = ProcessOrder(order);
    Assert.That(result.IsSuccess, Is.True);
}
```

### 9. Use SetUp for Common Initialization

```csharp
[TestFixture]
public class OrderServiceTests
{
    private OrderService sut;
    private FakeOrderRepository repository;
    private FakePaymentService paymentService;

    [SetUp]
    public void SetUp()
    {
        repository = new FakeOrderRepository();
        paymentService = new FakePaymentService();
        sut = new OrderService(repository, paymentService);
    }

    [Test]
    public void ProcessOrder_ValidOrder_SavesOrder()
    {
        var order = CreateOrder();

        sut.ProcessOrder(order);

        Assert.That(repository.SavedOrders, Has.Member(order));
    }
}
```

### 10. Test Exceptions Properly

```csharp
// ✅ Assert exception is thrown
[Test]
public void CreateUser_WithNullEmail_ThrowsArgumentNullException()
{
    Assert.Throws<ArgumentNullException>(() =>
    {
        new User(email: null!);
    });
}

// ✅ Assert exception message
[Test]
public void CreateUser_WithInvalidEmail_ThrowsWithMessage()
{
    var ex = Assert.Throws<ArgumentException>(() =>
    {
        new User(email: "invalid");
    });

    Assert.That(ex.Message, Does.Contain("Invalid email format"));
}
```

### 11. Use Parameterized Tests

```csharp
[TestCase("john@example.com", true)]
[TestCase("invalid.email", false)]
[TestCase("", false)]
[TestCase(null, false)]
public void IsValidEmail_VariousInputs_ReturnsExpected(
    string email,
    bool expected)
{
    var result = EmailValidator.IsValid(email);
    Assert.That(result, Is.EqualTo(expected));
}
```

### 12. Don't Test Framework Code

```csharp
// ❌ Testing LINQ (framework code)
[Test]
public void TestLinqSum()
{
    var numbers = new[] { 1, 2, 3 };
    var sum = numbers.Sum();
    Assert.That(sum, Is.EqualTo(6));
}

// ✅ Test your domain logic
[Test]
public void CalculateOrderTotal_MultipleItems_SumsCorrectly()
{
    var order = new OrderBuilder()
        .WithItem(price: 10m, quantity: 2)
        .WithItem(price: 5m, quantity: 3)
        .Build();

    var total = order.CalculateTotal();

    Assert.That(total, Is.EqualTo(35m));
}
```

## Symptoms

- Tests break when refactoring
- Unclear test failures
- Tests that don't catch regressions
- Slow test suite
- Tests that test the framework

## Benefits

- **Catch bugs early** with comprehensive tests
- **Safe refactoring** with test coverage
- **Documentation** through test examples
- **Design feedback** from testability
- **Confidence** in changes

## See Also

- [Test Data Builders](./test-data-builders.md) — Building test objects
- [Arrange-Act-Assert](../testing/arrange-act-assert.md) — Test structure
- [Test Doubles](../testing/test-doubles.md) — Mocks, fakes, stubs
