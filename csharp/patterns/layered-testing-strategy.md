# Layered Testing Strategy

> Test each architectural layer with appropriate strategies—unit tests for domain logic, integration tests for infrastructure, and end-to-end tests for critical paths—maximizing confidence while minimizing test maintenance.

## Problem

When testing strategies don't align with architectural layers, tests become slow, brittle, and expensive to maintain. Testing domain logic through HTTP endpoints makes tests fragile and slow.

## Core Concept

Match testing strategy to architectural layer:

```
┌─────────────────────────────────────────────────┐
│  Presentation (Controllers, HTTP)               │
│  Test: Few E2E tests for critical paths         │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│  Application (Use Cases, Commands, Queries)     │
│  Test: Integration tests with fake repos        │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│  Domain (Entities, Value Objects, Services)     │
│  Test: Many fast unit tests                     │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│  Infrastructure (Repositories, Adapters)        │
│  Test: Integration tests with real database     │
└─────────────────────────────────────────────────┘
```

**Test Pyramid:**
- 70% Unit Tests (Domain layer)
- 20% Integration Tests (Application + Infrastructure)
- 10% End-to-End Tests (Full stack)

## Example

### Domain Layer Tests (Unit Tests)

```csharp
// Tests/Domain/OrderTests.cs
public class OrderTests
{
    [Fact]
    public void Order_AddItem_WithValidProduct_AddsItem()
    {
        // ✅ Pure unit test - no dependencies
        var order = Order.Create(new CustomerId(Guid.NewGuid()));
        var product = Product.Create("Widget", Money.USD(10), stock: 100);
        
        var result = order.AddItem(product, quantity: 5);
        
        Assert.True(result.IsSuccess);
        Assert.Single(order.Items);
        Assert.Equal(Money.USD(50), order.Total);
    }
    
    [Fact]
    public void Order_AddItem_WithInsufficientStock_ReturnsError()
    {
        var order = Order.Create(new CustomerId(Guid.NewGuid()));
        var product = Product.Create("Widget", Money.USD(10), stock: 3);
        
        var result = order.AddItem(product, quantity: 5);
        
        Assert.False(result.IsSuccess);
        Assert.Equal("Insufficient stock", result.Error);
    }
    
    [Fact]
    public void Order_Submit_WithNoItems_ReturnsError()
    {
        var order = Order.Create(new CustomerId(Guid.NewGuid()));
        
        var result = order.Submit();
        
        Assert.False(result.IsSuccess);
        Assert.Equal("Cannot submit empty order", result.Error);
    }
}

// Tests/Domain/PricingServiceTests.cs
public class PricingServiceTests
{
    [Fact]
    public void CalculateDiscount_ForBulkOrder_AppliesBulkDiscount()
    {
        var service = new PricingService();
        var order = CreateOrderWithTotal(Money.USD(150));
        var customer = CreateCustomer(isPremium: false);
        
        var discount = service.CalculateDiscount(order, customer);
        
        Assert.Equal(Money.USD(15), discount);  // 10% of 150
    }
}
```

### Application Layer Tests (Integration Tests with Fakes)

```csharp
// Tests/Application/PlaceOrderUseCaseTests.cs
public class PlaceOrderUseCaseTests
{
    [Fact]
    public async Task PlaceOrder_WithValidItems_CreatesOrder()
    {
        // ✅ Integration test with fake repositories
        var fakeOrderRepo = new FakeOrderRepository();
        var fakeProductRepo = new FakeProductRepository();
        var fakeUnitOfWork = new FakeUnitOfWork();
        var fakeEmailService = new FakeEmailService();
        
        var product = Product.Create("Widget", Money.USD(10), stock: 100);
        fakeProductRepo.Add(product);
        
        var useCase = new PlaceOrderUseCase(
            fakeOrderRepo,
            fakeProductRepo,
            fakeUnitOfWork,
            new PricingService(),
            new FraudDetectionService(),
            fakeEmailService);
        
        var command = new PlaceOrderCommand(
            new CustomerId(Guid.NewGuid()),
            new List<OrderItemCommand>
            {
                new(product.Id, 5)
            });
        
        var result = await useCase.ExecuteAsync(command);
        
        Assert.True(result.IsSuccess);
        Assert.Single(fakeOrderRepo.Orders);
        Assert.True(fakeUnitOfWork.WasCommitted);
        Assert.Single(fakeEmailService.SentEmails);
    }
}

// Test doubles
public class FakeOrderRepository : IOrderRepository
{
    public List<Order> Orders { get; } = new();
    
    public Task<Option<Order>> GetByIdAsync(OrderId id)
    {
        var order = Orders.FirstOrDefault(o => o.Id == id);
        return Task.FromResult(order != null 
            ? Option<Order>.Some(order)
            : Option<Order>.None);
    }
    
    public Task SaveAsync(Order order)
    {
        Orders.Add(order);
        return Task.CompletedTask;
    }
}
```

### Infrastructure Layer Tests (Integration Tests with Real Database)

```csharp
// Tests/Infrastructure/OrderRepositoryTests.cs
public class OrderRepositoryTests : IDisposable
{
    private readonly DbConnection _connection;
    
    public OrderRepositoryTests()
    {
        // ✅ Use test database
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();
        
        // Create schema
        var command = _connection.CreateCommand();
        command.CommandText = @"
            CREATE TABLE Orders (
                Id TEXT PRIMARY KEY,
                CustomerId TEXT NOT NULL,
                Total REAL NOT NULL,
                Status TEXT NOT NULL
            );";
        command.ExecuteNonQuery();
    }
    
    [Fact]
    public async Task SaveAsync_WithOrder_PersistsToDatabase()
    {
        var repository = new SqlOrderRepository(_connection);
        var order = Order.Create(new CustomerId(Guid.NewGuid()));
        
        await repository.SaveAsync(order);
        
        var retrieved = await repository.GetByIdAsync(order.Id);
        Assert.True(retrieved.IsSome);
        retrieved.MatchSome(o => Assert.Equal(order.Id, o.Id));
    }
    
    public void Dispose()
    {
        _connection.Dispose();
    }
}
```

### End-to-End Tests (Full Stack)

```csharp
// Tests/EndToEnd/PlaceOrderE2ETests.cs
public class PlaceOrderE2ETests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    public PlaceOrderE2ETests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }
    
    [Fact]
    public async Task PlaceOrder_FullFlow_ReturnsCreated()
    {
        // ✅ Full stack test - HTTP → Controller → Use Case → Repository → Database
        var client = _factory.CreateClient();
        
        var request = new
        {
            CustomerId = Guid.NewGuid(),
            Items = new[]
            {
                new { ProductId = Guid.NewGuid(), Quantity = 2 }
            }
        };
        
        var response = await client.PostAsJsonAsync("/api/orders", request);
        
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        
        var result = await response.Content.ReadFromJsonAsync<OrderResponse>();
        Assert.NotNull(result);
        Assert.NotEqual(Guid.Empty, result.OrderId);
    }
}
```

## Test Distribution

| Layer | Test Type | Count | Speed | Confidence |
|-------|-----------|-------|-------|------------|
| Domain | Unit | 70% | Fast (ms) | High for business logic |
| Application | Integration (fakes) | 15% | Medium (100ms) | High for workflows |
| Infrastructure | Integration (real DB) | 10% | Slow (seconds) | High for data access |
| End-to-End | Full stack | 5% | Very slow (seconds) | High for critical paths |

## Benefits

1. **Fast Feedback**: Most tests are fast unit tests
2. **Targeted**: Test each layer with appropriate strategy
3. **Maintainable**: Fewer brittle E2E tests
4. **Confidence**: Right tests in right places

## See Also

- [Clean Architecture](./clean-architecture.md) — layer separation
- [Domain Model Isolation](./domain-model-isolation.md) — pure domain
