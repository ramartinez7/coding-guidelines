# Test Categories and Traits

> Organize tests with categories and traits—run specific subsets of tests based on speed, environment, or purpose.

## Problem

Test suites grow large and slow. Running all tests takes too long during development. You need to run only relevant tests—unit tests during development, integration tests before commits, slow tests in CI.

## Pattern

Use test categories and traits to organize tests into logical groups. Run test subsets based on what you're working on or the environment you're in.

## Example

### ❌ Before - No Organization

```csharp
[Fact]
public void CalculateDiscount_ValidInput_ReturnsCorrectValue() { }

[Fact]
public void SaveToDatabase_ValidOrder_PersistsCorrectly() { }  // Slow, needs DB

[Fact]
public void CallExternalApi_WithTimeout_HandlesGracefully() { }  // Slow, needs network

[Fact]
public void ValidateEmail_InvalidFormat_ReturnsFalse() { }
```

**Problem**: `dotnet test` runs everything, including slow tests.

### ✅ After - Organized with Traits

```csharp
[Fact]
[Trait("Category", "Unit")]
[Trait("Speed", "Fast")]
public void CalculateDiscount_ValidInput_ReturnsCorrectValue() { }

[Fact]
[Trait("Category", "Integration")]
[Trait("Speed", "Slow")]
[Trait("Requires", "Database")]
public void SaveToDatabase_ValidOrder_PersistsCorrectly() { }

[Fact]
[Trait("Category", "Integration")]
[Trait("Speed", "Slow")]
[Trait("Requires", "Network")]
public void CallExternalApi_WithTimeout_HandlesGracefully() { }

[Fact]
[Trait("Category", "Unit")]
[Trait("Speed", "Fast")]
public void ValidateEmail_InvalidFormat_ReturnsFalse() { }
```

**Run specific tests:**

```bash
# Run only fast tests during development
dotnet test --filter "Speed=Fast"

# Run only unit tests
dotnet test --filter "Category=Unit"

# Run tests that don't need database
dotnet test --filter "Requires!=Database"
```

## Using MSTest Categories

### MSTest TestCategory Attribute

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class OrderServiceTests
{
    [TestMethod]
    [TestCategory("Unit")]
    [TestCategory("Fast")]
    public void CalculateTotal_MultipleItems_SumsCorrectly()
    {
        // Arrange
        var items = new[]
        {
            OrderItem.Create("PROD-1", 2, Money.USD(10m)),
            OrderItem.Create("PROD-2", 1, Money.USD(15m))
        };

        // Act
        var total = OrderService.CalculateTotal(items);

        // Assert
        total.Should().Be(Money.USD(35m));
    }

    [TestMethod]
    [TestCategory("Integration")]
    [TestCategory("Slow")]
    [TestCategory("Database")]
    public void SaveOrder_ValidOrder_PersistsToDatabase()
    {
        // Arrange
        using var context = CreateTestDatabase();
        var service = new OrderService(context);
        var order = CreateValidOrder();

        // Act
        service.SaveOrder(order);

        // Assert
        var saved = context.Orders.Find(order.Id);
        saved.Should().NotBeNull();
    }
}
```

**Run by category:**

```bash
# MSTest
dotnet test --filter "TestCategory=Unit"
dotnet test --filter "TestCategory!=Integration"
dotnet test --filter "TestCategory=Fast&TestCategory=Unit"
```

## Custom Trait Attributes

### Creating Custom Attributes

```csharp
using Xunit.Sdk;

[TraitDiscoverer("CategoryDiscoverer", "YourAssembly")]
[AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
public class CategoryAttribute : Attribute, ITraitAttribute
{
    public CategoryAttribute(string category)
    {
        Category = category;
    }

    public string Category { get; }
}

public class CategoryDiscoverer : ITraitDiscoverer
{
    public IEnumerable<KeyValuePair<string, string>> GetTraits(IAttributeInfo traitAttribute)
    {
        var category = traitAttribute.GetNamedArgument<string>("Category");
        yield return new KeyValuePair<string, string>("Category", category);
    }
}
```

### Semantic Category Attributes

```csharp
// Speed categories
public class FastTestAttribute : TraitAttribute
{
    public FastTestAttribute() : base("Speed", "Fast") { }
}

public class SlowTestAttribute : TraitAttribute
{
    public SlowTestAttribute() : base("Speed", "Slow") { }
}

// Type categories
public class UnitTestAttribute : TraitAttribute
{
    public UnitTestAttribute() : base("Category", "Unit") { }
}

public class IntegrationTestAttribute : TraitAttribute
{
    public IntegrationTestAttribute() : base("Category", "Integration") { }
}

// Dependency categories
public class RequiresDatabaseAttribute : TraitAttribute
{
    public RequiresDatabaseAttribute() : base("Requires", "Database") { }
}

public class RequiresNetworkAttribute : TraitAttribute
{
    public RequiresNetworkAttribute() : base("Requires", "Network") { }
}

// Layer categories
public class DomainTestAttribute : TraitAttribute
{
    public DomainTestAttribute() : base("Layer", "Domain") { }
}

public class InfrastructureTestAttribute : TraitAttribute
{
    public InfrastructureTestAttribute() : base("Layer", "Infrastructure") { }
}
```

### Using Custom Attributes

```csharp
[Fact]
[UnitTest]
[FastTest]
[DomainTest]
public void CreateOrder_ValidInput_ReturnsSuccess()
{
    // Arrange
    var customerId = CustomerId.New();
    var items = CreateOrderItems();

    // Act
    var result = Order.Create(customerId, items);

    // Assert
    result.Should().BeSuccess();
}

[Fact]
[IntegrationTest]
[SlowTest]
[RequiresDatabase]
[InfrastructureTest]
public void SaveOrder_ToDatabase_PersistsCorrectly()
{
    // Arrange
    using var context = CreateTestDatabase();
    var repository = new OrderRepository(context);
    var order = CreateValidOrder();

    // Act
    repository.Save(order);

    // Assert
    var saved = repository.GetById(order.Id);
    saved.Should().BeEquivalentTo(order);
}
```

## Test Organization Strategies

### By Speed

```csharp
// Fast: < 100ms
[Fact]
[FastTest]
public void PureFunction_Returns_ExpectedResult() { }

// Medium: 100ms - 1s
[Fact]
[Trait("Speed", "Medium")]
public void WithFileAccess_Returns_ExpectedResult() { }

// Slow: > 1s
[Fact]
[SlowTest]
public void WithDatabaseAccess_Returns_ExpectedResult() { }
```

### By Layer

```csharp
[Fact]
[DomainTest]
public void DomainLogic_ValidInput_EnforcesInvariant() { }

[Fact]
[Trait("Layer", "Application")]
public void ApplicationService_Orchestrates_UseCaseCorrectly() { }

[Fact]
[InfrastructureTest]
public void Repository_Persists_DataCorrectly() { }

[Fact]
[Trait("Layer", "API")]
public void Controller_Returns_ExpectedResponse() { }
```

### By Feature

```csharp
[Fact]
[Trait("Feature", "OrderProcessing")]
public void ProcessOrder_ValidOrder_Succeeds() { }

[Fact]
[Trait("Feature", "Payment")]
public void ProcessPayment_ValidCard_CompletesTransaction() { }

[Fact]
[Trait("Feature", "Shipping")]
public void CalculateShipping_ValidAddress_ReturnsRate() { }
```

### By Priority

```csharp
[Fact]
[Trait("Priority", "Critical")]
[Trait("SmokeTest", "true")]
public void SystemStartup_Succeeds() { }

[Fact]
[Trait("Priority", "High")]
public void CoreBusinessLogic_WorksCorrectly() { }

[Fact]
[Trait("Priority", "Low")]
public void EdgeCase_HandledCorrectly() { }
```

## Filter Expressions

### Running Specific Test Subsets

```bash
# Run fast unit tests
dotnet test --filter "Category=Unit&Speed=Fast"

# Run integration tests that don't need database
dotnet test --filter "Category=Integration&Requires!=Database"

# Run all tests except slow ones
dotnet test --filter "Speed!=Slow"

# Run domain or application layer tests
dotnet test --filter "Layer=Domain|Layer=Application"

# Run critical priority tests
dotnet test --filter "Priority=Critical"

# Run smoke tests
dotnet test --filter "SmokeTest=true"

# Run specific feature tests
dotnet test --filter "Feature=OrderProcessing"

# Complex filters
dotnet test --filter "(Category=Unit&Speed=Fast)|(Priority=Critical)"
```

## CI/CD Pipeline Organization

### Different Test Stages

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run Unit Tests
        run: dotnet test --filter "Category=Unit" --logger "trx"

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v2
      - name: Setup Database
        run: docker-compose up -d database
      - name: Run Integration Tests
        run: dotnet test --filter "Category=Integration" --logger "trx"

  smoke-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    steps:
      - uses: actions/checkout@v2
      - name: Run Smoke Tests
        run: dotnet test --filter "SmokeTest=true" --logger "trx"
```

## Test Configuration Files

### .runsettings File

```xml
<?xml version="1.0" encoding="utf-8"?>
<RunSettings>
  <RunConfiguration>
    <MaxCpuCount>4</MaxCpuCount>
  </RunConfiguration>
  
  <MSTest>
    <SettingsFile>Local.testsettings</SettingsFile>
  </MSTest>
  
  <TestRunParameters>
    <Parameter name="TestCategory" value="Unit" />
  </TestRunParameters>
</RunSettings>
```

**Use with:**

```bash
dotnet test --settings Local.runsettings
```

## Directory Structure Organization

### Organizing Test Projects

```
Tests/
├── Unit/
│   ├── Domain/
│   │   └── OrderTests.cs         [UnitTest, DomainTest, FastTest]
│   └── Application/
│       └── OrderServiceTests.cs  [UnitTest, FastTest]
├── Integration/
│   ├── Database/
│   │   └── OrderRepositoryTests.cs [IntegrationTest, RequiresDatabase]
│   └── Api/
│       └── OrderControllerTests.cs [IntegrationTest, RequiresNetwork]
└── EndToEnd/
    └── OrderWorkflowTests.cs      [E2ETest, SlowTest]
```

## Conditional Test Execution

### Skip Tests Based on Environment

```csharp
public class ConditionalTestAttribute : FactAttribute
{
    public ConditionalTestAttribute(string environmentVariable, string expectedValue)
    {
        var actualValue = Environment.GetEnvironmentVariable(environmentVariable);
        
        if (actualValue != expectedValue)
        {
            Skip = $"Skipped because {environmentVariable}={actualValue}, expected {expectedValue}";
        }
    }
}

[ConditionalTest("CI", "true")]
public void ExpensiveTest_OnlyInCI_Runs()
{
    // Only runs in CI environment
}
```

## Guidelines

1. **Consistent naming**: Agree on category names across team
2. **Multiple traits**: Apply multiple traits for flexible filtering
3. **Document filters**: Create README with common filter commands
4. **CI optimization**: Run fast tests first, slow tests later
5. **Local development**: Default to fast tests, opt-in to slow tests

## Common Category Schemes

### Test Pyramid Categories

```csharp
[UnitTest]           // Base of pyramid - most tests
[IntegrationTest]    // Middle of pyramid - fewer tests
[E2ETest]            // Top of pyramid - fewest tests
```

### By Dependencies

```csharp
[Isolated]           // No external dependencies
[RequiresDatabase]   // Needs database
[RequiresNetwork]    // Needs network/API
[RequiresFileSystem] // Needs file I/O
```

### By Test Type

```csharp
[SmokeTest]          // Basic functionality
[RegressionTest]     // Prevent previous bugs
[PerformanceTest]    // Performance requirements
[SecurityTest]       // Security requirements
```

## Benefits

1. **Faster feedback**: Run relevant tests during development
2. **CI optimization**: Staged test execution in pipelines
3. **Selective execution**: Run tests for specific features
4. **Resource management**: Skip tests needing unavailable resources
5. **Clear organization**: Tests categorized by purpose and requirements

## See Also

- [Test Isolation and Independence](./test-isolation-independence.md) — independent test execution
- [Integration Test Patterns](./integration-test-patterns.md) — testing with external dependencies
- [Testing Async Code](./testing-async-code.md) — async test performance
- [Property-Based Testing](./property-based-testing.md) — expensive generative tests
