# Test Isolation and Independence

> Tests should not depend on each other—run in any order with consistent results.

## Problem

Tests that share state or depend on execution order create brittle test suites. Tests pass when run in one order but fail in another. Debugging becomes a nightmare when one test's failure cascades to others.

## Example

### ❌ Before - Shared State

```csharp
public class OrderServiceTests
{
    // ❌ Shared mutable state
    private static OrderRepository sharedRepository = new OrderRepository();
    private static Order testOrder;

    [Fact]
    public void Test1_CreateOrder()
    {
        testOrder = Order.Create(CustomerId.New(), Money.USD(100m));
        sharedRepository.Add(testOrder);
        Assert.NotNull(testOrder);
    }

    [Fact]
    public void Test2_ProcessOrder()
    {
        // ❌ Depends on Test1 running first
        var result = sharedRepository.Process(testOrder.Id);
        Assert.True(result.IsSuccess);
    }

    [Fact]
    public void Test3_GetOrder()
    {
        // ❌ Depends on Test1 and Test2
        var order = sharedRepository.Get(testOrder.Id);
        Assert.Equal(OrderStatus.Processed, order.Status);
    }
}
```

### ✅ After - Isolated Tests

```csharp
public class OrderServiceTests
{
    [Fact]
    public void CreateOrder_WithValidInputs_SavesOrderToPendingState()
    {
        // Arrange
        var repository = new InMemoryOrderRepository();
        var customerId = CustomerId.New();
        var amount = Money.USD(100m);

        // Act
        var order = Order.Create(customerId, amount);
        repository.Add(order);

        // Assert
        var saved = repository.Get(order.Id);
        Assert.NotNull(saved);
        Assert.Equal(OrderStatus.Pending, saved.Status);
    }

    [Fact]
    public void ProcessOrder_WithValidOrder_UpdatesStatusToProcessed()
    {
        // Arrange
        var repository = new InMemoryOrderRepository();
        var order = Order.Create(CustomerId.New(), Money.USD(100m));
        repository.Add(order);

        // Act
        var result = repository.Process(order.Id);

        // Assert
        Assert.True(result.IsSuccess);
        var processed = repository.Get(order.Id);
        Assert.Equal(OrderStatus.Processed, processed.Status);
    }

    [Fact]
    public void GetOrder_AfterProcessing_ReturnsProcessedOrder()
    {
        // Arrange
        var repository = new InMemoryOrderRepository();
        var order = Order.Create(CustomerId.New(), Money.USD(100m));
        repository.Add(order);
        repository.Process(order.Id);

        // Act
        var retrieved = repository.Get(order.Id);

        // Assert
        Assert.NotNull(retrieved);
        Assert.Equal(OrderStatus.Processed, retrieved.Status);
    }
}
```

## Principles

### 1. Fresh State Per Test

Each test should set up its own state, run, and clean up after itself.

```csharp
[Fact]
public void Test()
{
    // Arrange - Create fresh instances
    var repository = new InMemoryOrderRepository();
    var service = new OrderService(repository);
    
    // Act
    var result = service.DoSomething();
    
    // Assert
    Assert.True(result.IsSuccess);
    
    // No explicit cleanup needed if using in-memory fakes
}
```

### 2. Constructor Setup for Common Dependencies

Use constructor or `IDisposable` pattern for shared setup that creates *new instances* per test.

```csharp
public class OrderServiceTests : IDisposable
{
    private readonly IOrderRepository repository;
    private readonly OrderService service;

    public OrderServiceTests()
    {
        // Fresh instances per test
        repository = new InMemoryOrderRepository();
        service = new OrderService(repository);
    }

    [Fact]
    public void Test1()
    {
        // Use fresh service instance
    }

    [Fact]
    public void Test2()
    {
        // Gets its own fresh service instance
    }

    public void Dispose()
    {
        // Cleanup if needed
    }
}
```

### 3. No Static Mutable State

```csharp
// ❌ Bad - Shared between tests
private static List<Order> orders = new List<Order>();

// ✅ Good - Instance per test
private List<Order> orders = new List<Order>();

// ✅ Better - Created fresh in each test
[Fact]
public void Test()
{
    var orders = new List<Order>();
}
```

### 4. No Dependencies Between Tests

```csharp
// ❌ Bad - Tests must run in order
[Fact, Priority(1)]
public void SetupData() { }

[Fact, Priority(2)]
public void UseData() { }

// ✅ Good - Each test is complete
[Fact]
public void ProcessOrder_WithValidOrder_Succeeds()
{
    // Setup everything needed for THIS test
}
```

## Test Fixtures for Integration Tests

When testing against real resources (databases, files), ensure proper isolation:

### ✅ Database Tests

```csharp
public class OrderRepositoryIntegrationTests : IDisposable
{
    private readonly DbConnection connection;
    private readonly OrderRepository repository;

    public OrderRepositoryIntegrationTests()
    {
        // Each test gets a new connection
        connection = new SqlConnection(TestConnectionString);
        connection.Open();
        
        // Use transactions for isolation
        var transaction = connection.BeginTransaction();
        repository = new OrderRepository(connection, transaction);
    }

    [Fact]
    public void SaveOrder_WithValidOrder_PersistsToDatabase()
    {
        // This test's changes will be rolled back
        var order = Order.Create(CustomerId.New(), Money.USD(100m));
        repository.Save(order);
        
        var retrieved = repository.Get(order.Id);
        Assert.NotNull(retrieved);
    }

    public void Dispose()
    {
        // Rollback transaction - no data persists
        connection?.Dispose();
    }
}
```

### ✅ File System Tests

```csharp
public class FileProcessorTests : IDisposable
{
    private readonly string testDirectory;

    public FileProcessorTests()
    {
        // Each test gets its own directory
        testDirectory = Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString());
        Directory.CreateDirectory(testDirectory);
    }

    [Fact]
    public void ProcessFile_WithValidFile_CreatesOutput()
    {
        var inputPath = Path.Combine(testDirectory, "input.txt");
        File.WriteAllText(inputPath, "test content");
        
        // Test in isolation
    }

    public void Dispose()
    {
        if (Directory.Exists(testDirectory))
            Directory.Delete(testDirectory, recursive: true);
    }
}
```

## Benefits

1. **Parallel execution**: Tests can run simultaneously
2. **Reliable results**: Order-independent, deterministic outcomes
3. **Easy debugging**: Failures are localized to single tests
4. **Fast feedback**: Can run tests in any subset
5. **Refactoring safety**: Changing one test doesn't break others

## Anti-Patterns

**Shared static state**
```csharp
private static int counter = 0; // ❌ Breaks isolation
```

**Test sequence dependencies**
```csharp
[Fact, TestOrder(1)]
public void CreateUser() { } // ❌ Wrong

[Fact, TestOrder(2)]
public void UseUser() { } // ❌ Depends on test 1
```

**Persistent test data**
```csharp
[OneTimeSetUp]
public void SetupDatabase()
{
    // ❌ Data lives across all tests
    database.Insert(testData);
}
```

## See Also

- [Arrange-Act-Assert Pattern](./arrange-act-assert.md) — structuring independent tests
- [Test Data Builders](./test-data-builders.md) — creating test data easily
- [Object Mother Pattern](./object-mother-pattern.md) — reusable test fixtures
- [Integration Test Patterns](./integration-test-patterns.md) — isolating integration tests
