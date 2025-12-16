# Testing Smart Constructors

> Verify validated object construction, factory methods that enforce invariants, and constructor guards using FluentAssertions.

## Problem

Smart constructors encapsulate validation logic and ensure objects are always created in valid states. Tests must verify all validation rules, error messages, and that invalid objects cannot be constructed.

## Pattern

Test smart constructors for valid construction, all validation failures, invariant enforcement, and immutability guarantees.

## Example

### ❌ Before - Public Constructor

```csharp
[TestMethod]
public void CreateMoney_Works()
{
    var money = new Money(-100m, "USD"); // ❌ Allows invalid state
    Assert.IsNotNull(money);
}
```

### ✅ After - Smart Constructor Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

public record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    private Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    public static Result<Money> Create(decimal amount, string currency)
    {
        if (amount < 0)
            return Result.Failure<Money>("Amount cannot be negative");

        if (string.IsNullOrWhiteSpace(currency))
            return Result.Failure<Money>("Currency is required");

        if (currency.Length != 3)
            return Result.Failure<Money>("Currency must be 3 characters (ISO 4217)");

        return Result.Success(new Money(amount, currency.ToUpperInvariant()));
    }

    // Named constructors for common currencies
    public static Money USD(decimal amount) => Create(amount, "USD").Value;
    public static Money EUR(decimal amount) => Create(amount, "EUR").Value;
    public static Money GBP(decimal amount) => Create(amount, "GBP").Value;
}

[TestClass]
public class MoneySmartConstructorTests
{
    [TestMethod]
    public void Create_WithValidData_ReturnsSuccess()
    {
        // Act
        var result = Money.Create(100.50m, "USD");

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Amount.Should().Be(100.50m);
        result.Value.Currency.Should().Be("USD");
    }

    [TestMethod]
    public void Create_NormalizeCurrency_ToUpperCase()
    {
        // Act
        var result = Money.Create(50m, "usd");

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Currency.Should().Be("USD");
    }

    [TestMethod]
    public void Create_WithNegativeAmount_ReturnsFailure()
    {
        // Act
        var result = Money.Create(-10m, "USD");

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("negative");
    }

    [TestMethod]
    public void Create_WithEmptyCurrency_ReturnsFailure()
    {
        // Act
        var result = Money.Create(100m, "");

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("required");
    }

    [TestMethod]
    public void Create_WithInvalidCurrencyLength_ReturnsFailure()
    {
        // Act
        var result = Money.Create(100m, "US");

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("3 characters");
    }

    [TestMethod]
    public void USD_CreatesUSDMoney()
    {
        // Act
        var money = Money.USD(75m);

        // Assert
        money.Amount.Should().Be(75m);
        money.Currency.Should().Be("USD");
    }

    [TestMethod]
    public void EUR_CreatesEURMoney()
    {
        // Act
        var money = Money.EUR(100m);

        // Assert
        money.Amount.Should().Be(100m);
        money.Currency.Should().Be("EUR");
    }
}
```

## Testing Multi-Step Validation

```csharp
public record Customer
{
    public CustomerId Id { get; }
    public string FirstName { get; }
    public string LastName { get; }
    public string Email { get; }
    public DateTime DateOfBirth { get; }

    private Customer(CustomerId id, string firstName, string lastName, string email, DateTime dateOfBirth)
    {
        Id = id;
        FirstName = firstName;
        LastName = lastName;
        Email = email;
        DateOfBirth = dateOfBirth;
    }

    public static Result<Customer> Create(
        string firstName,
        string lastName,
        string email,
        DateTime dateOfBirth)
    {
        // Validate first name
        if (string.IsNullOrWhiteSpace(firstName))
            return Result.Failure<Customer>("First name is required");

        if (firstName.Length > 50)
            return Result.Failure<Customer>("First name cannot exceed 50 characters");

        // Validate last name
        if (string.IsNullOrWhiteSpace(lastName))
            return Result.Failure<Customer>("Last name is required");

        if (lastName.Length > 50)
            return Result.Failure<Customer>("Last name cannot exceed 50 characters");

        // Validate email
        if (string.IsNullOrWhiteSpace(email))
            return Result.Failure<Customer>("Email is required");

        if (!email.Contains("@"))
            return Result.Failure<Customer>("Email must be valid");

        // Validate date of birth
        if (dateOfBirth >= DateTime.Now)
            return Result.Failure<Customer>("Date of birth must be in the past");

        var age = DateTime.Now.Year - dateOfBirth.Year;
        if (age < 18)
            return Result.Failure<Customer>("Customer must be at least 18 years old");

        return Result.Success(new Customer(
            CustomerId.New(),
            firstName.Trim(),
            lastName.Trim(),
            email.ToLowerInvariant().Trim(),
            dateOfBirth
        ));
    }
}

[TestClass]
public class CustomerSmartConstructorTests
{
    [TestMethod]
    public void Create_WithAllValidData_ReturnsSuccess()
    {
        // Arrange
        var dob = DateTime.Now.AddYears(-30);

        // Act
        var result = Customer.Create("John", "Doe", "john@example.com", dob);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.FirstName.Should().Be("John");
        result.Value.LastName.Should().Be("Doe");
        result.Value.Email.Should().Be("john@example.com");
        result.Value.Id.Should().NotBeNull();
    }

    [TestMethod]
    public void Create_TrimsWhitespace()
    {
        // Arrange
        var dob = DateTime.Now.AddYears(-30);

        // Act
        var result = Customer.Create("  John  ", "  Doe  ", "  john@example.com  ", dob);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.FirstName.Should().Be("John");
        result.Value.LastName.Should().Be("Doe");
        result.Value.Email.Should().Be("john@example.com");
    }

    [TestMethod]
    public void Create_NormalizesEmail()
    {
        // Arrange
        var dob = DateTime.Now.AddYears(-30);

        // Act
        var result = Customer.Create("John", "Doe", "JOHN@EXAMPLE.COM", dob);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Email.Should().Be("john@example.com");
    }

    [TestMethod]
    public void Create_WithEmptyFirstName_ReturnsFailure()
    {
        // Arrange
        var dob = DateTime.Now.AddYears(-30);

        // Act
        var result = Customer.Create("", "Doe", "john@example.com", dob);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("First name");
    }

    [TestMethod]
    public void Create_WithTooLongFirstName_ReturnsFailure()
    {
        // Arrange
        var longName = new string('a', 51);
        var dob = DateTime.Now.AddYears(-30);

        // Act
        var result = Customer.Create(longName, "Doe", "john@example.com", dob);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("50 characters");
    }

    [TestMethod]
    public void Create_WithInvalidEmail_ReturnsFailure()
    {
        // Arrange
        var dob = DateTime.Now.AddYears(-30);

        // Act
        var result = Customer.Create("John", "Doe", "invalid-email", dob);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("valid");
    }

    [TestMethod]
    public void Create_WithFutureDateOfBirth_ReturnsFailure()
    {
        // Arrange
        var futureDob = DateTime.Now.AddYears(1);

        // Act
        var result = Customer.Create("John", "Doe", "john@example.com", futureDob);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("past");
    }

    [TestMethod]
    public void Create_WithUnderageCustomer_ReturnsFailure()
    {
        // Arrange
        var dob = DateTime.Now.AddYears(-10);

        // Act
        var result = Customer.Create("John", "Doe", "john@example.com", dob);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("18 years old");
    }
}
```

## Testing Invariant Preservation

```csharp
public record OrderLine
{
    public ProductId ProductId { get; }
    public int Quantity { get; }
    public Money UnitPrice { get; }
    public Money Total { get; }

    private OrderLine(ProductId productId, int quantity, Money unitPrice)
    {
        ProductId = productId;
        Quantity = quantity;
        UnitPrice = unitPrice;
        Total = Money.Create(unitPrice.Amount * quantity, unitPrice.Currency).Value;
    }

    public static Result<OrderLine> Create(ProductId productId, int quantity, Money unitPrice)
    {
        if (quantity <= 0)
            return Result.Failure<OrderLine>("Quantity must be positive");

        if (quantity > 10000)
            return Result.Failure<OrderLine>("Quantity cannot exceed 10,000");

        if (unitPrice.Amount <= 0)
            return Result.Failure<OrderLine>("Unit price must be positive");

        return Result.Success(new OrderLine(productId, quantity, unitPrice));
    }

    // Immutable update
    public Result<OrderLine> WithQuantity(int newQuantity)
    {
        return Create(ProductId, newQuantity, UnitPrice);
    }
}

[TestClass]
public class OrderLineInvariantTests
{
    [TestMethod]
    public void Create_CalculatesTotalCorrectly()
    {
        // Arrange
        var productId = ProductId.New();
        var quantity = 5;
        var unitPrice = Money.USD(10m);

        // Act
        var result = OrderLine.Create(productId, quantity, unitPrice);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Total.Amount.Should().Be(50m);
        result.Value.Total.Currency.Should().Be("USD");
    }

    [TestMethod]
    public void WithQuantity_RecalculatesTotal()
    {
        // Arrange
        var productId = ProductId.New();
        var orderLine = OrderLine.Create(productId, 5, Money.USD(10m)).Value;

        // Act
        var result = orderLine.WithQuantity(10);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Total.Amount.Should().Be(100m);
        result.Value.Quantity.Should().Be(10);
    }

    [TestMethod]
    public void WithQuantity_PreservesImmutability()
    {
        // Arrange
        var productId = ProductId.New();
        var original = OrderLine.Create(productId, 5, Money.USD(10m)).Value;

        // Act
        var result = original.WithQuantity(10);

        // Assert
        original.Quantity.Should().Be(5);
        original.Total.Amount.Should().Be(50m);
        result.Value.Should().NotBeSameAs(original);
    }

    [TestMethod]
    public void Create_WithZeroQuantity_ReturnsFailure()
    {
        // Act
        var result = OrderLine.Create(ProductId.New(), 0, Money.USD(10m));

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("positive");
    }

    [TestMethod]
    public void Create_WithExcessiveQuantity_ReturnsFailure()
    {
        // Act
        var result = OrderLine.Create(ProductId.New(), 10001, Money.USD(10m));

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("10,000");
    }
}
```

## Testing Dependent Validations

```csharp
public record DateRange
{
    public DateTime Start { get; }
    public DateTime End { get; }
    public TimeSpan Duration => End - Start;

    private DateRange(DateTime start, DateTime end)
    {
        Start = start;
        End = end;
    }

    public static Result<DateRange> Create(DateTime start, DateTime end)
    {
        if (start >= end)
            return Result.Failure<DateRange>("End date must be after start date");

        var duration = end - start;
        if (duration.TotalDays > 365)
            return Result.Failure<DateRange>("Date range cannot exceed 1 year");

        return Result.Success(new DateRange(start, end));
    }
}

[TestClass]
public class DependentValidationTests
{
    [TestMethod]
    public void Create_WithValidRange_ReturnsSuccess()
    {
        // Arrange
        var start = DateTime.Now;
        var end = start.AddDays(30);

        // Act
        var result = DateRange.Create(start, end);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Start.Should().Be(start);
        result.Value.End.Should().Be(end);
        result.Value.Duration.Should().Be(TimeSpan.FromDays(30));
    }

    [TestMethod]
    public void Create_WithEndBeforeStart_ReturnsFailure()
    {
        // Arrange
        var start = DateTime.Now;
        var end = start.AddDays(-1);

        // Act
        var result = DateRange.Create(start, end);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("after start");
    }

    [TestMethod]
    public void Create_WithEqualDates_ReturnsFailure()
    {
        // Arrange
        var date = DateTime.Now;

        // Act
        var result = DateRange.Create(date, date);

        // Assert
        result.IsFailure.Should().BeTrue();
    }

    [TestMethod]
    public void Create_WithExcessiveRange_ReturnsFailure()
    {
        // Arrange
        var start = DateTime.Now;
        var end = start.AddDays(366);

        // Act
        var result = DateRange.Create(start, end);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("1 year");
    }
}
```

## Why It's Important

1. **Invariant Enforcement**: Invalid objects cannot exist
2. **Single Responsibility**: Validation in one place
3. **Fail Fast**: Errors caught at construction
4. **Type Safety**: Compiler prevents misuse
5. **Self-Documenting**: Construction shows requirements

## Guidelines

**Testing Smart Constructors**
- Test successful construction with valid data
- Test each validation rule independently
- Test boundary conditions
- Test normalization (trimming, case)
- Test invariant calculations
- Test immutability via update methods

**Validation Testing**
- Test all failure paths
- Verify error messages
- Test dependent validations
- Test edge cases

**Common Patterns**
- `.IsSuccess.Should().BeTrue()`
- `.IsFailure.Should().BeTrue()`
- `.Error.Should().Contain("text")`
- Verify all properties set correctly
- Test immutability

## Common Pitfalls

❌ **Not testing all validations**
```csharp
// Incomplete - missing other validations
Customer.Create("", "Doe", "john@example.com", dob)
    .IsFailure.Should().BeTrue();
```

✅ **Test each validation**
```csharp
// Complete
Customer.Create("", "Doe", "email", dob).IsFailure.Should().BeTrue();
Customer.Create("John", "", "email", dob).IsFailure.Should().BeTrue();
Customer.Create("John", "Doe", "", dob).IsFailure.Should().BeTrue();
Customer.Create("John", "Doe", "email", future).IsFailure.Should().BeTrue();
```

❌ **Not verifying normalization**
```csharp
// Missing normalization check
var result = Money.Create(100m, "usd");
result.IsSuccess.Should().BeTrue();
```

✅ **Verify normalized values**
```csharp
// Complete
var result = Money.Create(100m, "usd");
result.IsSuccess.Should().BeTrue();
result.Value.Currency.Should().Be("USD"); // Normalized
```

## See Also

- [Testing Refinement Types](./testing-refinement-types.md) — constrained types
- [Testing Guard Clauses](./testing-guard-clauses.md) — validation guards
- [Smart Constructors](../patterns/smart-constructors.md) — construction pattern
- [Static Factory Methods](../patterns/static-factory-methods.md) — factory methods
