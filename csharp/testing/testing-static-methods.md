# Testing Static Methods

> Test static methods effectively—verify pure functions, manage static state, and handle singleton patterns.

## Problem

Static methods can't be easily mocked and may have hidden dependencies. Tests must verify behavior while managing static state and side effects.

## Pattern

Treat static methods as pure functions when possible. For stateful static methods, reset state between tests and use dependency injection where appropriate.

## Example

### ❌ Before - Untestable Static State

```csharp
public static class ConfigurationManager
{
    private static Dictionary<string, string> settings = new();

    public static void SetSetting(string key, string value)
    {
        settings[key] = value;  // Shared mutable state
    }

    public static string GetSetting(string key)
    {
        return settings.TryGetValue(key, out var value) ? value : "";
    }
}

// Tests interfere with each other due to shared state
```

### ✅ After - Testable Static Methods

```csharp
public static class ConfigurationManager
{
    private static Dictionary<string, string> settings = new();

    public static void SetSetting(string key, string value)
    {
        settings[key] = value;
    }

    public static string GetSetting(string key)
    {
        return settings.TryGetValue(key, out var value) ? value : "";
    }

    // Test helper - not for production use
    internal static void Reset()
    {
        settings = new Dictionary<string, string>();
    }
}

[Fact]
public void GetSetting_AfterSet_ReturnsValue()
{
    // Arrange
    ConfigurationManager.Reset();  // Clean slate
    ConfigurationManager.SetSetting("key1", "value1");

    // Act
    var result = ConfigurationManager.GetSetting("key1");

    // Assert
    result.Should().Be("value1");
    
    // Cleanup
    ConfigurationManager.Reset();
}
```

## Testing Pure Static Functions

### Mathematical Functions

```csharp
public static class MathHelper
{
    public static decimal CalculateCompoundInterest(
        decimal principal,
        decimal rate,
        int periods)
    {
        return principal * (decimal)Math.Pow((double)(1 + rate), periods);
    }

    public static bool IsPrime(int number)
    {
        if (number <= 1) return false;
        if (number == 2) return true;
        if (number % 2 == 0) return false;

        var sqrt = (int)Math.Sqrt(number);
        for (int i = 3; i <= sqrt; i += 2)
        {
            if (number % i == 0) return false;
        }
        return true;
    }
}

[Theory]
[InlineData(1000, 0.05, 10, 1628.89)]
[InlineData(5000, 0.03, 5, 5796.37)]
public void CalculateCompoundInterest_VariousInputs_ReturnsCorrectAmount(
    decimal principal,
    decimal rate,
    int periods,
    decimal expected)
{
    // Act
    var result = MathHelper.CalculateCompoundInterest(principal, rate, periods);

    // Assert
    result.Should().BeApproximately(expected, 0.01m);
}

[Theory]
[InlineData(2, true)]
[InlineData(3, true)]
[InlineData(4, false)]
[InlineData(17, true)]
[InlineData(20, false)]
public void IsPrime_VariousNumbers_IdentifiesCorrectly(int number, bool expected)
{
    // Act
    var result = MathHelper.IsPrime(number);

    // Assert
    result.Should().Be(expected);
}
```

## Testing Static Factory Methods

### Object Creation

```csharp
public record EmailAddress
{
    private EmailAddress(string value) => Value = value;

    public string Value { get; }

    public static Result<EmailAddress> Create(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            return Result.Failure<EmailAddress>("Email is required");

        if (!email.Contains("@"))
            return Result.Failure<EmailAddress>("Invalid email format");

        return Result.Success(new EmailAddress(email));
    }
}

[Theory]
[InlineData("user@example.com", true)]
[InlineData("invalid", false)]
[InlineData("", false)]
[InlineData(null, false)]
public void Create_VariousInputs_ValidatesCorrectly(string? email, bool shouldSucceed)
{
    // Act
    var result = EmailAddress.Create(email!);

    // Assert
    result.IsSuccess.Should().Be(shouldSucceed);

    if (shouldSucceed)
    {
        result.Value.Value.Should().Be(email);
    }
}
```

## Testing Static Utility Classes

### String Utilities

```csharp
public static class StringUtils
{
    public static string ToSlug(string text)
    {
        if (string.IsNullOrWhiteSpace(text))
            return string.Empty;

        return text
            .ToLowerInvariant()
            .Replace(" ", "-")
            .Replace("_", "-");
    }

    public static string Truncate(string text, int maxLength, string suffix = "...")
    {
        if (string.IsNullOrEmpty(text) || text.Length <= maxLength)
            return text ?? string.Empty;

        return text[..(maxLength - suffix.Length)] + suffix;
    }
}

[Theory]
[InlineData("Hello World", "hello-world")]
[InlineData("Test_Value", "test-value")]
[InlineData("", "")]
[InlineData(null, "")]
public void ToSlug_VariousInputs_ConvertsCorrectly(string? input, string expected)
{
    // Act
    var result = StringUtils.ToSlug(input!);

    // Assert
    result.Should().Be(expected);
}

[Theory]
[InlineData("Hello World", 5, "...", "He...")]
[InlineData("Short", 10, "...", "Short")]
[InlineData("Long text here", 8, "...", "Long ...")]
public void Truncate_VariousInputs_TruncatesCorrectly(
    string input,
    int maxLength,
    string suffix,
    string expected)
{
    // Act
    var result = StringUtils.Truncate(input, maxLength, suffix);

    // Assert
    result.Should().Be(expected);
}
```

## Testing Static State Management

### Singleton Pattern

```csharp
public class DatabaseConnection
{
    private static DatabaseConnection? instance;
    private static readonly object lockObject = new();

    private DatabaseConnection() { }

    public static DatabaseConnection Instance
    {
        get
        {
            if (instance == null)
            {
                lock (lockObject)
                {
                    if (instance == null)
                    {
                        instance = new DatabaseConnection();
                    }
                }
            }
            return instance;
        }
    }

    // For testing only
    internal static void ResetInstance()
    {
        instance = null;
    }
}

[Fact]
public void Instance_CalledMultipleTimes_ReturnsSameInstance()
{
    // Arrange
    DatabaseConnection.ResetInstance();

    // Act
    var instance1 = DatabaseConnection.Instance;
    var instance2 = DatabaseConnection.Instance;

    // Assert
    instance1.Should().BeSameAs(instance2);

    // Cleanup
    DatabaseConnection.ResetInstance();
}

[Fact]
public void Instance_ThreadSafe_CreatesSingleInstance()
{
    // Arrange
    DatabaseConnection.ResetInstance();
    var instances = new ConcurrentBag<DatabaseConnection>();

    // Act
    Parallel.For(0, 100, _ =>
    {
        instances.Add(DatabaseConnection.Instance);
    });

    // Assert
    instances.Distinct().Should().HaveCount(1);

    // Cleanup
    DatabaseConnection.ResetInstance();
}
```

## Testing Static Caching

### Cache Management

```csharp
public static class CurrencyCache
{
    private static readonly Dictionary<string, decimal> exchangeRates = new();
    private static readonly object lockObject = new();

    public static void SetRate(string currency, decimal rate)
    {
        lock (lockObject)
        {
            exchangeRates[currency] = rate;
        }
    }

    public static decimal GetRate(string currency)
    {
        lock (lockObject)
        {
            return exchangeRates.TryGetValue(currency, out var rate) ? rate : 1.0m;
        }
    }

    public static void Clear()
    {
        lock (lockObject)
        {
            exchangeRates.Clear();
        }
    }
}

[Fact]
public void SetRate_NewCurrency_StoresRate()
{
    // Arrange
    CurrencyCache.Clear();

    // Act
    CurrencyCache.SetRate("EUR", 1.18m);
    var rate = CurrencyCache.GetRate("EUR");

    // Assert
    rate.Should().Be(1.18m);

    // Cleanup
    CurrencyCache.Clear();
}

[Fact]
public void GetRate_UnknownCurrency_ReturnsDefault()
{
    // Arrange
    CurrencyCache.Clear();

    // Act
    var rate = CurrencyCache.GetRate("UNKNOWN");

    // Assert
    rate.Should().Be(1.0m);

    // Cleanup
    CurrencyCache.Clear();
}
```

## Testing Static Extension Points

### Strategy Pattern with Static Methods

```csharp
public static class Validator
{
    private static Func<string, bool> customValidator = _ => true;

    public static void SetCustomValidator(Func<string, bool> validator)
    {
        customValidator = validator;
    }

    public static bool Validate(string input)
    {
        return !string.IsNullOrWhiteSpace(input) && customValidator(input);
    }

    internal static void ResetValidator()
    {
        customValidator = _ => true;
    }
}

[Fact]
public void Validate_WithCustomValidator_UsesCustomLogic()
{
    // Arrange
    Validator.ResetValidator();
    Validator.SetCustomValidator(input => input.Length >= 5);

    // Act
    var validResult = Validator.Validate("Hello");
    var invalidResult = Validator.Validate("Hi");

    // Assert
    validResult.Should().BeTrue();
    invalidResult.Should().BeFalse();

    // Cleanup
    Validator.ResetValidator();
}
```

## Testing Static Constructors

### Initialization Logic

```csharp
public static class Configuration
{
    public static readonly string DefaultConnectionString;
    public static readonly int MaxRetries;

    static Configuration()
    {
        DefaultConnectionString = Environment.GetEnvironmentVariable("DB_CONNECTION")
            ?? "Server=localhost";
        MaxRetries = int.Parse(Environment.GetEnvironmentVariable("MAX_RETRIES") ?? "3");
    }
}

[Fact]
public void StaticConstructor_LoadsConfiguration()
{
    // Static constructor has already run
    
    // Assert
    Configuration.DefaultConnectionString.Should().NotBeNullOrEmpty();
    Configuration.MaxRetries.Should().BeGreaterThan(0);
}
```

## Wrapping Static Methods for Testing

### Abstraction Layer

```csharp
// Problematic static class
public static class Clock
{
    public static DateTime UtcNow => DateTime.UtcNow;
}

// Testable abstraction
public interface ITimeProvider
{
    DateTime UtcNow { get; }
}

public class SystemTimeProvider : ITimeProvider
{
    public DateTime UtcNow => DateTime.UtcNow;
}

public class FixedTimeProvider : ITimeProvider
{
    private readonly DateTime fixedTime;

    public FixedTimeProvider(DateTime fixedTime)
    {
        this.fixedTime = fixedTime;
    }

    public DateTime UtcNow => fixedTime;
}

[Fact]
public void CreateOrder_UsesCurrentTime()
{
    // Arrange
    var fixedTime = new DateTime(2024, 1, 1, 12, 0, 0, DateTimeKind.Utc);
    var timeProvider = new FixedTimeProvider(fixedTime);
    var orderService = new OrderService(timeProvider);

    // Act
    var order = orderService.CreateOrder(CustomerId.New(), CreateItems());

    // Assert
    order.CreatedAt.Should().Be(fixedTime);
}
```

## Guidelines

1. **Prefer pure functions**: Static methods without side effects are easiest to test
2. **Reset state**: Clean up static state between tests
3. **Thread safety**: Test concurrent access to static state
4. **Use abstractions**: Wrap problematic static methods with testable interfaces
5. **Document side effects**: Clearly indicate when static methods have side effects

## Benefits

1. **Testable utilities**: Verify helper functions work correctly
2. **Isolated tests**: Reset static state prevents test interference
3. **Thread safety**: Tests verify concurrent access works correctly
4. **Clear contracts**: Tests document static method behavior
5. **Refactoring support**: Tests enable safe changes to static methods

## Common Pitfalls

### Shared State Interference

```csharp
// ❌ Tests interfere
[Fact]
public void Test1() 
{ 
    ConfigManager.Set("key", "value1");
    // ...
}

[Fact]
public void Test2()  
{ 
    // May see "value1" from Test1
    var value = ConfigManager.Get("key");
}

// ✅ Tests isolated
[Fact]
public void Test1() 
{ 
    ConfigManager.Reset();
    ConfigManager.Set("key", "value1");
    // ...
}

[Fact]
public void Test2()
{ 
    ConfigManager.Reset();
    var value = ConfigManager.Get("key");
    // ...
}
```

## See Also

- [Testing Factory Methods](./testing-factory-methods.md) — testing static factory methods
- [Testing Concurrency](./testing-concurrency.md) — testing thread-safe static methods
- [Testing Guard Clauses](./testing-guard-clauses.md) — testing static validation
- [Testing Singletons](./testing-lazy-initialization.md) — testing singleton patterns
