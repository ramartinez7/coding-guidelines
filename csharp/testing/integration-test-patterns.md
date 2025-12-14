# Integration Test Patterns

> Test components working together—balance speed, reliability, and realistic scenarios.

## Problem

Unit tests verify components in isolation, but integration issues often occur when components interact. Full end-to-end tests are slow and brittle. Integration tests bridge this gap.

## Pattern

Test key integration points with real dependencies (databases, file systems, message queues) while keeping tests fast and maintainable through careful scoping and test data management.

## Example

### ❌ Before - Missing Integration Tests

```csharp
// Only unit tests with mocks
[Fact]
public void SaveOrder_CallsRepository()
{
    var mockRepo = new Mock<IOrderRepository>();
    var service = new OrderService(mockRepo.Object);
    
    service.Save(order);
    
    mockRepo.Verify(x => x.Save(It.IsAny<Order>()), Times.Once);
}
// ❌ Doesn't test actual database interaction!
```

### ✅ After - Integration Test

```csharp
public class OrderRepositoryIntegrationTests : IDisposable
{
    private readonly DbConnection connection;
    private readonly DbTransaction transaction;
    private readonly OrderRepository repository;

    public OrderRepositoryIntegrationTests()
    {
        connection = new SqlConnection(TestConnectionString);
        connection.Open();
        transaction = connection.BeginTransaction();
        repository = new OrderRepository(connection, transaction);
    }

    [Fact]
    public async Task SaveOrder_PersistsToDatabase_AndCanBeRetrieved()
    {
        // Arrange
        var order = OrderBuilder.Default().Build();

        // Act
        await repository.SaveAsync(order);
        var retrieved = await repository.GetAsync(order.Id);

        // Assert
        Assert.NotNull(retrieved);
        Assert.Equal(order.Id, retrieved.Id);
        Assert.Equal(order.CustomerId, retrieved.CustomerId);
        Assert.Equal(order.Total, retrieved.Total);
    }

    public void Dispose()
    {
        transaction?.Rollback(); // Undo changes
        transaction?.Dispose();
        connection?.Dispose();
    }
}
```

## Database Integration Tests

### Using Transactions for Isolation

```csharp
public class DatabaseTestFixture : IDisposable
{
    public DbConnection Connection { get; }
    public DbTransaction Transaction { get; }

    public DatabaseTestFixture()
    {
        Connection = new SqlConnection(GetTestConnectionString());
        Connection.Open();
        Transaction = Connection.BeginTransaction();
    }

    private static string GetTestConnectionString()
    {
        return Environment.GetEnvironmentVariable("TEST_DB_CONNECTION") 
            ?? "Server=localhost;Database=TestDb;Integrated Security=true;";
    }

    public void Dispose()
    {
        Transaction?.Rollback();
        Transaction?.Dispose();
        Connection?.Dispose();
    }
}

public class OrderRepositoryTests : IClassFixture<DatabaseTestFixture>
{
    private readonly DatabaseTestFixture fixture;
    private readonly OrderRepository repository;

    public OrderRepositoryTests(DatabaseTestFixture fixture)
    {
        this.fixture = fixture;
        this.repository = new OrderRepository(fixture.Connection, fixture.Transaction);
    }

    [Fact]
    public async Task SaveAndRetrieve_Order_RoundTripsCorrectly()
    {
        // Arrange
        var order = OrderBuilder.Default()
            .WithCustomerId(CustomerId.New())
            .WithTotal(Money.USD(150m))
            .Build();

        // Act
        await repository.SaveAsync(order);
        var retrieved = await repository.GetAsync(order.Id);

        // Assert
        Assert.NotNull(retrieved);
        Assert.Equal(order.Total, retrieved.Total);
    }
}
```

### Using Test Containers

```csharp
// Using Testcontainers library
public class DatabaseContainerFixture : IAsyncLifetime
{
    private PostgreSqlContainer? container;
    public string ConnectionString { get; private set; } = string.Empty;

    public async Task InitializeAsync()
    {
        container = new PostgreSqlBuilder()
            .WithImage("postgres:15")
            .WithDatabase("testdb")
            .WithUsername("test")
            .WithPassword("test")
            .Build();

        await container.StartAsync();
        ConnectionString = container.GetConnectionString();

        // Run migrations
        await RunMigrations(ConnectionString);
    }

    public async Task DisposeAsync()
    {
        if (container != null)
            await container.DisposeAsync();
    }

    private async Task RunMigrations(string connectionString)
    {
        // Apply database schema
    }
}

[Collection("Database")]
public class OrderRepositoryContainerTests : IClassFixture<DatabaseContainerFixture>
{
    private readonly string connectionString;

    public OrderRepositoryContainerTests(DatabaseContainerFixture fixture)
    {
        this.connectionString = fixture.ConnectionString;
    }

    [Fact]
    public async Task SaveOrder_ToRealDatabase_Succeeds()
    {
        await using var connection = new NpgsqlConnection(connectionString);
        await connection.OpenAsync();
        
        await using var transaction = await connection.BeginTransactionAsync();
        var repository = new OrderRepository(connection, transaction);

        var order = OrderBuilder.Default().Build();
        
        await repository.SaveAsync(order);
        var retrieved = await repository.GetAsync(order.Id);

        Assert.NotNull(retrieved);
        Assert.Equal(order.Id, retrieved.Id);
        
        await transaction.RollbackAsync();
    }
}
```

## File System Integration Tests

```csharp
public class FileProcessorIntegrationTests : IDisposable
{
    private readonly string testDirectory;

    public FileProcessorIntegrationTests()
    {
        testDirectory = Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString());
        Directory.CreateDirectory(testDirectory);
    }

    [Fact]
    public async Task ProcessFile_CreatesOutputFile()
    {
        // Arrange
        var inputPath = Path.Combine(testDirectory, "input.csv");
        var outputPath = Path.Combine(testDirectory, "output.json");
        
        await File.WriteAllTextAsync(inputPath, "id,name,price\n1,Widget,10.00");
        
        var processor = new FileProcessor();

        // Act
        await processor.ProcessAsync(inputPath, outputPath);

        // Assert
        Assert.True(File.Exists(outputPath));
        var content = await File.ReadAllTextAsync(outputPath);
        Assert.Contains("Widget", content);
    }

    public void Dispose()
    {
        if (Directory.Exists(testDirectory))
            Directory.Delete(testDirectory, recursive: true);
    }
}
```

## HTTP API Integration Tests

### Using WebApplicationFactory

```csharp
public class OrderApiIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> factory;
    private readonly HttpClient client;

    public OrderApiIntegrationTests(WebApplicationFactory<Program> factory)
    {
        this.factory = factory;
        this.client = factory.CreateClient();
    }

    [Fact]
    public async Task CreateOrder_ReturnsCreatedOrder()
    {
        // Arrange
        var request = new CreateOrderRequest
        {
            CustomerId = CustomerId.New().ToString(),
            Items = new[]
            {
                new OrderItemRequest { ProductId = ProductId.New().ToString(), Quantity = 2 }
            }
        };

        // Act
        var response = await client.PostAsJsonAsync("/api/orders", request);

        // Assert
        response.EnsureSuccessStatusCode();
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        
        var createdOrder = await response.Content.ReadFromJsonAsync<OrderResponse>();
        Assert.NotNull(createdOrder);
        Assert.NotNull(createdOrder.OrderId);
    }

    [Fact]
    public async Task GetOrder_ExistingOrder_ReturnsOrder()
    {
        // Arrange - Create order first
        var createRequest = new CreateOrderRequest { /* ... */ };
        var createResponse = await client.PostAsJsonAsync("/api/orders", createRequest);
        var created = await createResponse.Content.ReadFromJsonAsync<OrderResponse>();

        // Act
        var response = await client.GetAsync($"/api/orders/{created!.OrderId}");

        // Assert
        response.EnsureSuccessStatusCode();
        var order = await response.Content.ReadFromJsonAsync<OrderResponse>();
        Assert.Equal(created.OrderId, order!.OrderId);
    }
}
```

### Custom Web Application Factory

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove real database
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));
            if (descriptor != null)
                services.Remove(descriptor);

            // Add in-memory database
            services.AddDbContext<ApplicationDbContext>(options =>
            {
                options.UseInMemoryDatabase("TestDb");
            });

            // Build service provider
            var sp = services.BuildServiceProvider();

            // Create database
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
            db.Database.EnsureCreated();
        });
    }
}
```

## Message Queue Integration Tests

```csharp
public class MessageQueueIntegrationTests : IAsyncLifetime
{
    private RabbitMqContainer? rabbitMqContainer;
    private string connectionString = string.Empty;

    public async Task InitializeAsync()
    {
        rabbitMqContainer = new RabbitMqBuilder()
            .WithImage("rabbitmq:3-management")
            .Build();

        await rabbitMqContainer.StartAsync();
        connectionString = rabbitMqContainer.GetConnectionString();
    }

    [Fact]
    public async Task PublishMessage_IsReceivedBySubscriber()
    {
        // Arrange
        var factory = new ConnectionFactory { Uri = new Uri(connectionString) };
        using var connection = factory.CreateConnection();
        using var channel = connection.CreateModel();

        var queueName = "test-queue";
        channel.QueueDeclare(queueName, durable: false, exclusive: true, autoDelete: true);

        var publisher = new MessagePublisher(connectionString);
        var receivedMessages = new List<OrderCreatedEvent>();

        // Subscribe
        var consumer = new EventingBasicConsumer(channel);
        consumer.Received += (model, ea) =>
        {
            var body = ea.Body.ToArray();
            var message = JsonSerializer.Deserialize<OrderCreatedEvent>(body);
            if (message != null)
                receivedMessages.Add(message);
        };
        channel.BasicConsume(queueName, autoAck: true, consumer);

        // Act
        var evt = new OrderCreatedEvent(OrderId.New(), CustomerId.New());
        await publisher.PublishAsync(evt);

        // Wait for message
        await Task.Delay(1000);

        // Assert
        Assert.Single(receivedMessages);
        Assert.Equal(evt.OrderId, receivedMessages[0].OrderId);
    }

    public async Task DisposeAsync()
    {
        if (rabbitMqContainer != null)
            await rabbitMqContainer.DisposeAsync();
    }
}
```

## Testing with Real External APIs

```csharp
[Trait("Category", "Integration")]
[Trait("Category", "External")]
public class PaymentGatewayIntegrationTests
{
    [Fact]
    [Skip("Requires payment gateway credentials")]
    public async Task ProcessPayment_WithTestCard_Succeeds()
    {
        // Arrange
        var apiKey = Environment.GetEnvironmentVariable("PAYMENT_API_KEY");
        if (string.IsNullOrEmpty(apiKey))
            throw new SkipException("Payment API key not configured");

        var gateway = new PaymentGateway(apiKey, useSandbox: true);
        var testCard = CardNumber.TestVisa();

        // Act
        var result = await gateway.ProcessPaymentAsync(testCard, Money.USD(1.00m));

        // Assert
        Assert.True(result.IsSuccess);
    }
}
```

## Testing Background Jobs

```csharp
public class BackgroundJobIntegrationTests
{
    [Fact]
    public async Task ProcessOrderJob_ProcessesPendingOrders()
    {
        // Arrange
        var repository = new InMemoryOrderRepository();
        var pendingOrder = OrderBuilder.Default()
            .WithStatus(OrderStatus.Pending)
            .Build();
        await repository.SaveAsync(pendingOrder);

        var job = new ProcessOrdersJob(repository);

        // Act
        await job.ExecuteAsync(CancellationToken.None);

        // Assert
        var processed = await repository.GetAsync(pendingOrder.Id);
        Assert.Equal(OrderStatus.Processing, processed.Status);
    }
}
```

## Guidelines for Integration Tests

### Test Pyramid Strategy

```
    /\
   /  \     E2E Tests (Few)
  /____\    
  /    \    Integration Tests (Some)
 /______\   
 /      \   Unit Tests (Many)
/_________\
```

### Best Practices

1. **Isolate tests**: Use transactions, separate databases, or containers
2. **Clean up**: Always rollback or delete test data
3. **Be selective**: Test critical integration points, not everything
4. **Keep them fast**: Use in-memory alternatives when possible
5. **Mark clearly**: Use `[Trait]` to identify integration tests
6. **Run separately**: Don't run with unit tests in CI

### Organizing Integration Tests

```csharp
// Separate test project or folder
IntegrationTests/
  Database/
    OrderRepositoryTests.cs
    CustomerRepositoryTests.cs
  Api/
    OrderApiTests.cs
    CustomerApiTests.cs
  External/
    PaymentGatewayTests.cs
    EmailServiceTests.cs
```

## Benefits

1. **Confidence**: Tests use real dependencies
2. **Find integration bugs**: Catch issues unit tests miss
3. **Realistic**: Test actual data flow
4. **Documentation**: Show how components interact
5. **Regression prevention**: Catch breaking changes

## Trade-offs

**Pros:**
- More realistic than unit tests
- Find integration issues early
- Test actual infrastructure

**Cons:**
- Slower than unit tests
- More complex setup
- Can be flaky
- May require external resources

## See Also

- [Test Isolation and Independence](./test-isolation-independence.md) — keeping tests isolated
- [Test Doubles](./test-doubles.md) — alternatives to real dependencies
- [Testing Async Code](./testing-async-code.md) — async integration tests
- [Testing Side Effects](./testing-side-effects.md) — verifying external interactions
