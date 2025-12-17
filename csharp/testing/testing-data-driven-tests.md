# Data-Driven Testing

> Load test data from external sources—CSV files, JSON, databases—to run comprehensive test suites without hardcoding values.

## Problem

Hardcoding test data in attributes limits flexibility and makes it hard to maintain large test datasets. Business users can't easily contribute test cases.

## Pattern

Externalize test data to files or databases, then load it dynamically. This separates test logic from test data and enables non-developers to contribute test cases.

## Example

### ❌ Before - Hardcoded Test Data

```csharp
[Theory]
[InlineData("user@example.com", true)]
[InlineData("invalid", false)]
[InlineData("test@test.com", true)]
[InlineData("@example.com", false)]
[InlineData("user@", false)]
// ... 50 more hardcoded cases
public void ValidateEmail_VariousFormats_ReturnsExpectedResult(string email, bool expected)
{
    var result = EmailValidator.IsValid(email);
    result.Should().Be(expected);
}
```

### ✅ After - Data-Driven from CSV

```csharp
public class EmailTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        var csvPath = Path.Combine("TestData", "email-validations.csv");
        
        foreach (var line in File.ReadLines(csvPath).Skip(1)) // Skip header
        {
            var parts = line.Split(',');
            var email = parts[0];
            var expected = bool.Parse(parts[1]);
            
            yield return new object[] { email, expected };
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(EmailTestData))]
public void ValidateEmail_FromCsv_ReturnsExpectedResult(string email, bool expected)
{
    // Act
    var result = EmailValidator.IsValid(email);

    // Assert
    result.Should().Be(expected);
}
```

**TestData/email-validations.csv:**

```csv
Email,IsValid
user@example.com,true
invalid,false
test@test.com,true
@example.com,false
user@,false
user@domain,false
very.long.email.address@subdomain.example.com,true
```

## CSV Data Source

### Helper Class for CSV Parsing

```csharp
public abstract class CsvDataSource : IEnumerable<object[]>
{
    protected abstract string CsvFileName { get; }
    protected abstract object[] ParseLine(string[] columns);

    public IEnumerator<object[]> GetEnumerator()
    {
        var csvPath = Path.Combine("TestData", CsvFileName);
        
        if (!File.Exists(csvPath))
            throw new FileNotFoundException($"Test data file not found: {csvPath}");

        var lines = File.ReadAllLines(csvPath);
        
        foreach (var line in lines.Skip(1)) // Skip header
        {
            if (string.IsNullOrWhiteSpace(line))
                continue;

            var columns = line.Split(',');
            yield return ParseLine(columns);
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

### Using CSV Data Source

```csharp
public class PriceCalculationData : CsvDataSource
{
    protected override string CsvFileName => "price-calculations.csv";

    protected override object[] ParseLine(string[] columns)
    {
        return new object[]
        {
            decimal.Parse(columns[0]),  // basePrice
            decimal.Parse(columns[1]),  // discountPercent
            decimal.Parse(columns[2])   // expectedPrice
        };
    }
}

[Theory]
[ClassData(typeof(PriceCalculationData))]
public void CalculatePrice_WithDiscount_ReturnsExpectedPrice(
    decimal basePrice, 
    decimal discountPercent, 
    decimal expectedPrice)
{
    // Arrange
    var calculator = new PriceCalculator();

    // Act
    var result = calculator.CalculateDiscountedPrice(basePrice, discountPercent);

    // Assert
    result.Should().BeApproximately(expectedPrice, 0.01m);
}
```

## JSON Data Source

### JSON Test Data File

**TestData/order-validations.json:**

```json
[
  {
    "customerId": "550e8400-e29b-41d4-a716-446655440000",
    "items": [
      { "productId": "PROD-001", "quantity": 2, "price": 10.00 }
    ],
    "expectedTotal": 20.00,
    "shouldSucceed": true
  },
  {
    "customerId": "00000000-0000-0000-0000-000000000000",
    "items": [],
    "expectedTotal": 0,
    "shouldSucceed": false
  }
]
```

### Loading JSON Data

```csharp
public record OrderTestCase(
    Guid CustomerId,
    List<OrderItem> Items,
    decimal ExpectedTotal,
    bool ShouldSucceed);

public record OrderItem(string ProductId, int Quantity, decimal Price);

public class OrderValidationData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        var jsonPath = Path.Combine("TestData", "order-validations.json");
        var json = File.ReadAllText(jsonPath);
        var testCases = JsonSerializer.Deserialize<List<OrderTestCase>>(json);

        foreach (var testCase in testCases!)
        {
            yield return new object[] 
            { 
                testCase.CustomerId,
                testCase.Items,
                testCase.ExpectedTotal,
                testCase.ShouldSucceed
            };
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(OrderValidationData))]
public void CreateOrder_VariousScenarios_ValidatesCorrectly(
    Guid customerId,
    List<OrderItem> items,
    decimal expectedTotal,
    bool shouldSucceed)
{
    // Arrange
    var orderItems = items.Select(i => 
        OrderLineItem.Create(i.ProductId, i.Quantity, Money.USD(i.Price))).ToList();

    // Act
    var result = Order.Create(CustomerId.From(customerId), orderItems);

    // Assert
    result.IsSuccess.Should().Be(shouldSucceed);
    
    if (shouldSucceed)
    {
        result.Value.Total.Should().Be(Money.USD(expectedTotal));
    }
}
```

## Database-Driven Tests

### Loading from Database

```csharp
public class DatabaseTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        var connectionString = "Data Source=test-data.db";
        
        using var connection = new SqliteConnection(connectionString);
        connection.Open();

        var command = connection.CreateCommand();
        command.CommandText = @"
            SELECT ProductCode, Description, ExpectedCategory
            FROM ProductTestCases
            WHERE IsActive = 1";

        using var reader = command.ExecuteReader();
        
        while (reader.Read())
        {
            yield return new object[]
            {
                reader.GetString(0),  // ProductCode
                reader.GetString(1),  // Description
                reader.GetString(2)   // ExpectedCategory
            };
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(DatabaseTestData))]
public void CategorizeProduct_FromDatabase_ReturnsCorrectCategory(
    string productCode,
    string description,
    string expectedCategory)
{
    // Arrange
    var categorizer = new ProductCategorizer();
    var product = new Product(productCode, description);

    // Act
    var category = categorizer.Categorize(product);

    // Assert
    category.Should().Be(expectedCategory);
}
```

## Excel Data Source

Using EPPlus or ClosedXML:

```csharp
public class ExcelTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        var excelPath = Path.Combine("TestData", "calculations.xlsx");
        
        using var package = new ExcelPackage(new FileInfo(excelPath));
        var worksheet = package.Workbook.Worksheets[0];
        
        var rowCount = worksheet.Dimension.Rows;
        
        // Start from row 2, skipping header
        for (int row = 2; row <= rowCount; row++)
        {
            var input1 = worksheet.Cells[row, 1].GetValue<decimal>();
            var input2 = worksheet.Cells[row, 2].GetValue<decimal>();
            var expected = worksheet.Cells[row, 3].GetValue<decimal>();
            
            yield return new object[] { input1, input2, expected };
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

## MemberData from Method

### Dynamic Test Generation

```csharp
public class DiscountCalculatorTests
{
    public static IEnumerable<object[]> GetDiscountTestCases()
    {
        // Load from multiple sources
        foreach (var testCase in LoadFromCsv())
            yield return testCase;
            
        foreach (var testCase in LoadFromJson())
            yield return testCase;
            
        // Add programmatic cases
        yield return new object[] { 100m, 0m, 100m };
        yield return new object[] { 100m, 10m, 90m };
        yield return new object[] { 100m, 100m, 0m };
    }

    private static IEnumerable<object[]> LoadFromCsv()
    {
        var csvPath = Path.Combine("TestData", "discounts.csv");
        
        if (!File.Exists(csvPath))
            yield break;

        foreach (var line in File.ReadLines(csvPath).Skip(1))
        {
            var parts = line.Split(',');
            yield return new object[]
            {
                decimal.Parse(parts[0]),
                decimal.Parse(parts[1]),
                decimal.Parse(parts[2])
            };
        }
    }

    private static IEnumerable<object[]> LoadFromJson()
    {
        var jsonPath = Path.Combine("TestData", "discounts.json");
        
        if (!File.Exists(jsonPath))
            yield break;

        var json = File.ReadAllText(jsonPath);
        var cases = JsonSerializer.Deserialize<List<DiscountTestCase>>(json);
        
        foreach (var c in cases!)
        {
            yield return new object[] { c.Price, c.Discount, c.Expected };
        }
    }

    [Theory]
    [MemberData(nameof(GetDiscountTestCases))]
    public void ApplyDiscount_VariousCases_ReturnsCorrectResult(
        decimal price,
        decimal discountPercent,
        decimal expected)
    {
        // Arrange
        var calculator = new DiscountCalculator();

        // Act
        var result = calculator.ApplyDiscount(price, discountPercent);

        // Assert
        result.Should().Be(expected);
    }
}
```

## Complex Domain Objects

### Loading Complex Test Scenarios

```csharp
public record OrderTestScenario(
    string ScenarioName,
    Customer Customer,
    List<Product> Products,
    ShippingAddress Address,
    PaymentMethod PaymentMethod,
    decimal ExpectedTotal,
    bool ShouldSucceed,
    string? ExpectedError);

public class OrderScenariosData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        var jsonPath = Path.Combine("TestData", "order-scenarios.json");
        var json = File.ReadAllText(jsonPath);
        
        var scenarios = JsonSerializer.Deserialize<List<OrderTestScenario>>(
            json,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

        foreach (var scenario in scenarios!)
        {
            yield return new object[] { scenario };
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(OrderScenariosData))]
public void ProcessOrder_ComplexScenarios_BehavesCorrectly(OrderTestScenario scenario)
{
    // Arrange
    var processor = new OrderProcessor();

    // Act
    var result = processor.Process(
        scenario.Customer,
        scenario.Products,
        scenario.Address,
        scenario.PaymentMethod);

    // Assert
    result.IsSuccess.Should().Be(scenario.ShouldSucceed);
    
    if (scenario.ShouldSucceed)
    {
        result.Value.Total.Should().Be(Money.USD(scenario.ExpectedTotal));
    }
    else
    {
        result.Error.Should().Contain(scenario.ExpectedError!);
    }
}
```

## Guidelines

1. **Version control test data**: Include test data files in repository
2. **Document format**: Add README explaining data file structure
3. **Validate on load**: Fail fast if data files are malformed
4. **Use meaningful names**: Name test cases descriptively in data files
5. **Keep data readable**: Choose formats business users can edit (CSV, Excel)
6. **Cache loaded data**: Don't reload files for every test

## Benefits

1. **Separation of concerns**: Test logic separate from test data
2. **Business collaboration**: Non-developers can add test cases
3. **Comprehensive coverage**: Easy to add many test cases
4. **Maintainability**: Update data without changing code
5. **Reusability**: Share test data across multiple test classes

## Performance Optimization

```csharp
// Cache loaded data to avoid repeated file I/O
public class CachedTestData : IEnumerable<object[]>
{
    private static List<object[]>? cachedData;
    private static readonly object lockObject = new object();

    public IEnumerator<object[]> GetEnumerator()
    {
        if (cachedData == null)
        {
            lock (lockObject)
            {
                if (cachedData == null)
                {
                    cachedData = LoadData().ToList();
                }
            }
        }

        return cachedData.GetEnumerator();
    }

    private IEnumerable<object[]> LoadData()
    {
        // Load from file...
        yield break;
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

## See Also

- [Parameterized Tests](./parameterized-tests.md) — running tests with different inputs
- [Object Mother Pattern](./object-mother-pattern.md) — creating standard test fixtures
- [Test Data Builders](./test-data-builders.md) — fluent test object creation
- [Property-Based Testing](./property-based-testing.md) — generated test data
