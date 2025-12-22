# Mutation Testing

> Using mutation testing to verify test quality—ensure your tests actually catch bugs by introducing deliberate defects.

## Problem

High code coverage doesn't guarantee quality tests. Tests might execute code without properly verifying behavior. You need confidence that tests will catch real bugs.

## Pattern

Mutation testing introduces small changes (mutations) to your code and verifies that tests fail. If tests still pass after a mutation, the tests aren't effectively verifying that behavior.

## Example

### ❌ Before - Weak Test

```csharp
public class Calculator
{
    public int Add(int a, int b)
    {
        return a + b;  // Mutation: Change to a - b
    }
}

[Fact]
public void Add_TwoNumbers_ReturnsValue()
{
    // Arrange
    var calculator = new Calculator();

    // Act
    var result = calculator.Add(5, 3);

    // Assert
    Assert.NotEqual(0, result);  // Weak assertion - would pass even with mutation
}
```

### ✅ After - Strong Test

```csharp
public class Calculator
{
    public int Add(int a, int b)
    {
        return a + b;
    }
}

[Fact]
public void Add_TwoNumbers_ReturnsCorrectSum()
{
    // Arrange
    var calculator = new Calculator();

    // Act
    var result = calculator.Add(5, 3);

    // Assert
    Assert.Equal(8, result);  // Strong assertion - will catch operator mutations
}

[Fact]
public void Add_ZeroToNumber_ReturnsNumber()
{
    // Arrange
    var calculator = new Calculator();

    // Act
    var result = calculator.Add(5, 0);

    // Assert
    Assert.Equal(5, result);  // Catches mutations to identity element
}
```

## Using Stryker.NET

Install: `dotnet tool install -g dotnet-stryker`

### Configuration

Create `stryker-config.json`:

```json
{
  "stryker-config": {
    "project": "YourProject.csproj",
    "test-projects": ["YourProject.Tests.csproj"],
    "reporters": ["html", "progress"],
    "thresholds": {
      "high": 80,
      "low": 60,
      "break": 50
    },
    "mutation-level": "complete"
  }
}
```

### Running Mutation Tests

```bash
# Run mutation testing
dotnet stryker

# Run with specific mutations
dotnet stryker --mutate "src/Domain/**/*.cs"

# Generate HTML report
dotnet stryker --reporter html
```

## Common Mutations

### Arithmetic Operators

```csharp
// Original
public Money Add(Money other) => new Money(this.amount + other.amount);

// Mutations to catch:
// + → -
// + → *
// + → /
```

**Test to catch mutations:**

```csharp
[Fact]
public void Add_TwoAmounts_ReturnsCorrectSum()
{
    // Arrange
    var money1 = Money.USD(10m);
    var money2 = Money.USD(5m);

    // Act
    var result = money1.Add(money2);

    // Assert
    result.Should().Be(Money.USD(15m));  // Catches +→- mutation
}
```

### Comparison Operators

```csharp
// Original
public bool IsPositive() => amount > 0;

// Mutations to catch:
// > → >=
// > → <
// > → <=
```

**Test to catch mutations:**

```csharp
[Theory]
[InlineData(1, true)]
[InlineData(0, false)]   // Catches > → >=
[InlineData(-1, false)]
public void IsPositive_ReturnsCorrectResult(decimal amount, bool expected)
{
    // Arrange
    var money = Money.USD(amount);

    // Act
    var result = money.IsPositive();

    // Assert
    result.Should().Be(expected);
}
```

### Boolean Logic

```csharp
// Original
public bool IsValid() => !string.IsNullOrEmpty(value) && value.Length <= 100;

// Mutations to catch:
// && → ||
// ! → removed
// <= → <
```

**Test to catch mutations:**

```csharp
[Theory]
[InlineData("valid", true)]
[InlineData("", false)]           // Catches !IsNullOrEmpty removal
[InlineData(null, false)]         // Catches && → ||
[InlineData("x" * 100, true)]     // Catches <= → <
[InlineData("x" * 101, false)]
public void IsValid_VariousInputs_ReturnsCorrectResult(string value, bool expected)
{
    // Arrange
    var validator = new StringValidator(value);

    // Act
    var result = validator.IsValid();

    // Assert
    result.Should().Be(expected);
}
```

### Return Values

```csharp
// Original
public Result<Order> CreateOrder()
{
    if (IsValid())
        return Result.Success(new Order());
    return Result.Failure<Order>("Invalid");
}

// Mutations to catch:
// Result.Success → Result.Failure
// Return value removed
```

**Test to catch mutations:**

```csharp
[Fact]
public void CreateOrder_WhenValid_ReturnsSuccessWithOrder()
{
    // Arrange
    var factory = ValidOrderFactory();

    // Act
    var result = factory.CreateOrder();

    // Assert
    result.IsSuccess.Should().BeTrue();
    result.Value.Should().NotBeNull();  // Catches return value mutations
}

[Fact]
public void CreateOrder_WhenInvalid_ReturnsFailure()
{
    // Arrange
    var factory = InvalidOrderFactory();

    // Act
    var result = factory.CreateOrder();

    // Assert
    result.IsFailure.Should().BeTrue();
    result.Error.Should().Contain("Invalid");
}
```

## Testing Domain Logic

### Value Object Validation

```csharp
public record EmailAddress
{
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
```

**Comprehensive tests to survive mutations:**

```csharp
[Theory]
[InlineData(null, false)]
[InlineData("", false)]
[InlineData("   ", false)]
[InlineData("no-at-sign", false)]
[InlineData("valid@email.com", true)]
public void Create_VariousInputs_ValidatesCorrectly(string input, bool shouldSucceed)
{
    // Act
    var result = EmailAddress.Create(input);

    // Assert
    result.IsSuccess.Should().Be(shouldSucceed);
    
    if (!shouldSucceed)
    {
        result.Error.Should().NotBeEmpty();
    }
}
```

## Mutation Score Goals

### Target Scores

```csharp
// Excellent: 80-100%
//   - Critical business logic
//   - Security-sensitive code
//   - Financial calculations

// Good: 60-80%
//   - Core domain logic
//   - Data transformations
//   - Validation rules

// Acceptable: 40-60%
//   - Infrastructure code
//   - Configuration
//   - Simple DTOs
```

## Handling Surviving Mutants

### Equivalent Mutants

Some mutations don't change behavior:

```csharp
// Original
for (int i = 0; i < items.Count; i++)
{
    Process(items[i]);
}

// Mutation: i++ → ++i
// This is equivalent - both work the same in this context
// Mark as ignored in stryker-config.json
```

### Add Missing Tests

```csharp
// Surviving mutant indicates missing boundary test
public bool IsAdult(int age) => age >= 18;

// Need test for exact boundary:
[Theory]
[InlineData(17, false)]
[InlineData(18, true)]  // Add this to catch >= → > mutation
public void IsAdult_AtBoundary_ReturnsCorrectResult(int age, bool expected)
{
    var result = AgeValidator.IsAdult(age);
    result.Should().Be(expected);
}
```

## Guidelines

1. **Run regularly**: Include in CI pipeline for critical code
2. **Focus on domain logic**: Don't mutate generated code or DTOs
3. **Improve weak tests**: Surviving mutants reveal assertion weaknesses
4. **Set realistic thresholds**: 100% mutation coverage is often impractical
5. **Combine with coverage**: Use both metrics together

## Benefits

1. **Test quality assurance**: Verify tests actually verify behavior
2. **Find weak assertions**: Identify tests that don't properly check results
3. **Missing edge cases**: Discover untested boundary conditions
4. **Confidence**: Know your tests will catch real bugs
5. **Better design**: Forces thinking about all code paths

## Performance Considerations

```csharp
// Mutation testing is slow - optimize by:

// 1. Test only changed files
dotnet stryker --since:main

// 2. Exclude infrastructure code
{
  "exclude": [
    "**/Migrations/**/*.cs",
    "**/Infrastructure/**/*.cs"
  ]
}

// 3. Run in parallel
{
  "concurrency": 4
}
```

## See Also

- [Testing Domain Invariants](./testing-domain-invariants.md) — verifying business rules
- [Property-Based Testing](./property-based-testing.md) — testing invariants
- [Testing Guard Clauses](./testing-guard-clauses.md) — input validation
- [One Assertion Per Test](./one-assertion-per-test.md) — focused verification
