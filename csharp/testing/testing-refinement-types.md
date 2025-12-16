# Testing Refinement Types

> Verify constrained types, validation rules, and type-level guarantees using MSTest and FluentAssertions.

## Problem

Refinement types enforce constraints at the type level, ensuring values always meet specific criteria. Tests must verify construction validation, invariants, and that invalid values cannot exist.

## Pattern

Test refinement type construction, validation rules, boundary conditions, and that types maintain their invariants.

## Example

### ❌ Before - Primitive with Manual Validation

```csharp
[TestMethod]
public void ProcessAge_Works()
{
    int age = 25;
    service.Process(age);
    // ❌ No guarantee age is valid
}
```

### ✅ After - Refinement Type Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

public record Age
{
    public int Value { get; }

    private Age(int value) => Value = value;

    public static Result<Age> Create(int value)
    {
        if (value < 0)
            return Result.Failure<Age>("Age cannot be negative");

        if (value > 150)
            return Result.Failure<Age>("Age cannot exceed 150");

        return Result.Success(new Age(value));
    }
}

[TestClass]
public class AgeRefinementTests
{
    [TestMethod]
    public void Create_WithValidAge_ReturnsSuccess()
    {
        // Act
        var result = Age.Create(25);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Value.Should().Be(25);
    }

    [TestMethod]
    public void Create_WithNegativeAge_ReturnsFailure()
    {
        // Act
        var result = Age.Create(-1);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("negative");
    }

    [TestMethod]
    public void Create_WithExcessiveAge_ReturnsFailure()
    {
        // Act
        var result = Age.Create(151);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("exceed");
    }

    [TestMethod]
    public void Create_WithBoundaryAge_ReturnsSuccess()
    {
        // Act
        var resultZero = Age.Create(0);
        var result150 = Age.Create(150);

        // Assert
        resultZero.IsSuccess.Should().BeTrue();
        result150.IsSuccess.Should().BeTrue();
    }
}
```

## Testing String Refinement Types

```csharp
public record NonEmptyString
{
    public string Value { get; }

    private NonEmptyString(string value) => Value = value;

    public static Result<NonEmptyString> Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return Result.Failure<NonEmptyString>("String cannot be empty or whitespace");

        return Result.Success(new NonEmptyString(value));
    }
}

public record BoundedString
{
    public string Value { get; }

    private BoundedString(string value) => Value = value;

    public static Result<BoundedString> Create(string value, int minLength, int maxLength)
    {
        if (string.IsNullOrWhiteSpace(value))
            return Result.Failure<BoundedString>("String cannot be empty");

        if (value.Length < minLength)
            return Result.Failure<BoundedString>($"String must be at least {minLength} characters");

        if (value.Length > maxLength)
            return Result.Failure<BoundedString>($"String cannot exceed {maxLength} characters");

        return Result.Success(new BoundedString(value));
    }
}

[TestClass]
public class StringRefinementTests
{
    [TestMethod]
    public void NonEmptyString_WithValidString_ReturnsSuccess()
    {
        // Act
        var result = NonEmptyString.Create("hello");

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Value.Should().Be("hello");
    }

    [TestMethod]
    public void NonEmptyString_WithEmptyString_ReturnsFailure()
    {
        // Act
        var result = NonEmptyString.Create("");

        // Assert
        result.IsFailure.Should().BeTrue();
    }

    [TestMethod]
    public void NonEmptyString_WithWhitespace_ReturnsFailure()
    {
        // Act
        var result = NonEmptyString.Create("   ");

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("whitespace");
    }

    [TestMethod]
    public void BoundedString_WithinBounds_ReturnsSuccess()
    {
        // Act
        var result = BoundedString.Create("hello", minLength: 3, maxLength: 10);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Value.Should().Be("hello");
    }

    [TestMethod]
    public void BoundedString_TooShort_ReturnsFailure()
    {
        // Act
        var result = BoundedString.Create("hi", minLength: 3, maxLength: 10);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("at least 3");
    }

    [TestMethod]
    public void BoundedString_TooLong_ReturnsFailure()
    {
        // Act
        var result = BoundedString.Create("hello world!", minLength: 3, maxLength: 10);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("cannot exceed 10");
    }
}
```

## Testing Numeric Refinement Types

```csharp
public record PositiveDecimal
{
    public decimal Value { get; }

    private PositiveDecimal(decimal value) => Value = value;

    public static Result<PositiveDecimal> Create(decimal value)
    {
        if (value <= 0)
            return Result.Failure<PositiveDecimal>("Value must be positive");

        return Result.Success(new PositiveDecimal(value));
    }
}

public record Percentage
{
    public decimal Value { get; }

    private Percentage(decimal value) => Value = value;

    public static Result<Percentage> Create(decimal value)
    {
        if (value < 0)
            return Result.Failure<Percentage>("Percentage cannot be negative");

        if (value > 100)
            return Result.Failure<Percentage>("Percentage cannot exceed 100");

        return Result.Success(new Percentage(value));
    }
}

[TestClass]
public class NumericRefinementTests
{
    [TestMethod]
    public void PositiveDecimal_WithPositiveValue_ReturnsSuccess()
    {
        // Act
        var result = PositiveDecimal.Create(42.5m);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Value.Should().Be(42.5m);
    }

    [TestMethod]
    public void PositiveDecimal_WithZero_ReturnsFailure()
    {
        // Act
        var result = PositiveDecimal.Create(0m);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("positive");
    }

    [TestMethod]
    public void PositiveDecimal_WithNegative_ReturnsFailure()
    {
        // Act
        var result = PositiveDecimal.Create(-10m);

        // Assert
        result.IsFailure.Should().BeTrue();
    }

    [TestMethod]
    public void Percentage_WithValidValue_ReturnsSuccess()
    {
        // Act
        var result = Percentage.Create(50m);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Value.Should().Be(50m);
    }

    [TestMethod]
    public void Percentage_WithBoundaryValues_ReturnsSuccess()
    {
        // Act
        var resultZero = Percentage.Create(0m);
        var result100 = Percentage.Create(100m);

        // Assert
        resultZero.IsSuccess.Should().BeTrue();
        result100.IsSuccess.Should().BeTrue();
    }

    [TestMethod]
    public void Percentage_AboveMax_ReturnsFailure()
    {
        // Act
        var result = Percentage.Create(101m);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("exceed 100");
    }
}
```

## Testing Pattern-Based Refinement Types

```csharp
public record EmailAddress
{
    public string Value { get; }

    private EmailAddress(string value) => Value = value;

    public static Result<EmailAddress> Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return Result.Failure<EmailAddress>("Email cannot be empty");

        if (!value.Contains("@"))
            return Result.Failure<EmailAddress>("Email must contain @");

        var parts = value.Split('@');
        if (parts.Length != 2)
            return Result.Failure<EmailAddress>("Email must have exactly one @");

        if (string.IsNullOrWhiteSpace(parts[0]) || string.IsNullOrWhiteSpace(parts[1]))
            return Result.Failure<EmailAddress>("Email must have local and domain parts");

        if (!parts[1].Contains("."))
            return Result.Failure<EmailAddress>("Domain must contain a dot");

        return Result.Success(new EmailAddress(value.ToLowerInvariant()));
    }
}

[TestClass]
public class PatternRefinementTests
{
    [TestMethod]
    public void EmailAddress_WithValidEmail_ReturnsSuccess()
    {
        // Act
        var result = EmailAddress.Create("user@example.com");

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Value.Should().Be("user@example.com");
    }

    [TestMethod]
    public void EmailAddress_NormalizesToLowerCase()
    {
        // Act
        var result = EmailAddress.Create("User@Example.COM");

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Value.Should().Be("user@example.com");
    }

    [TestMethod]
    public void EmailAddress_WithoutAtSign_ReturnsFailure()
    {
        // Act
        var result = EmailAddress.Create("userexample.com");

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("@");
    }

    [TestMethod]
    public void EmailAddress_WithMultipleAtSigns_ReturnsFailure()
    {
        // Act
        var result = EmailAddress.Create("user@@example.com");

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("exactly one");
    }

    [TestMethod]
    public void EmailAddress_WithoutDomainDot_ReturnsFailure()
    {
        // Act
        var result = EmailAddress.Create("user@examplecom");

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("dot");
    }
}
```

## Testing Composite Refinement Types

```csharp
public record ValidatedCustomer
{
    public NonEmptyString FirstName { get; }
    public NonEmptyString LastName { get; }
    public EmailAddress Email { get; }
    public Age Age { get; }

    private ValidatedCustomer(
        NonEmptyString firstName,
        NonEmptyString lastName,
        EmailAddress email,
        Age age)
    {
        FirstName = firstName;
        LastName = lastName;
        Email = email;
        Age = age;
    }

    public static Result<ValidatedCustomer> Create(
        string firstName,
        string lastName,
        string email,
        int age)
    {
        var firstNameResult = NonEmptyString.Create(firstName);
        if (firstNameResult.IsFailure)
            return Result.Failure<ValidatedCustomer>(firstNameResult.Error);

        var lastNameResult = NonEmptyString.Create(lastName);
        if (lastNameResult.IsFailure)
            return Result.Failure<ValidatedCustomer>(lastNameResult.Error);

        var emailResult = EmailAddress.Create(email);
        if (emailResult.IsFailure)
            return Result.Failure<ValidatedCustomer>(emailResult.Error);

        var ageResult = Age.Create(age);
        if (ageResult.IsFailure)
            return Result.Failure<ValidatedCustomer>(ageResult.Error);

        return Result.Success(new ValidatedCustomer(
            firstNameResult.Value,
            lastNameResult.Value,
            emailResult.Value,
            ageResult.Value));
    }
}

[TestClass]
public class CompositeRefinementTests
{
    [TestMethod]
    public void Create_WithAllValidData_ReturnsSuccess()
    {
        // Act
        var result = ValidatedCustomer.Create(
            "John",
            "Doe",
            "john@example.com",
            25
        );

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.FirstName.Value.Should().Be("John");
        result.Value.Email.Value.Should().Be("john@example.com");
        result.Value.Age.Value.Should().Be(25);
    }

    [TestMethod]
    public void Create_WithInvalidFirstName_ReturnsFailure()
    {
        // Act
        var result = ValidatedCustomer.Create(
            "",
            "Doe",
            "john@example.com",
            25
        );

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("empty");
    }

    [TestMethod]
    public void Create_WithInvalidEmail_ReturnsFailure()
    {
        // Act
        var result = ValidatedCustomer.Create(
            "John",
            "Doe",
            "invalid-email",
            25
        );

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("@");
    }

    [TestMethod]
    public void Create_WithInvalidAge_ReturnsFailure()
    {
        // Act
        var result = ValidatedCustomer.Create(
            "John",
            "Doe",
            "john@example.com",
            -5
        );

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("negative");
    }
}
```

## Testing Refinement Type Operations

```csharp
public static class AgeOperations
{
    public static Result<Age> Add(Age age1, Age age2)
    {
        var sum = age1.Value + age2.Value;
        return Age.Create(sum);
    }
}

[TestClass]
public class RefinementOperationTests
{
    [TestMethod]
    public void Add_ValidAges_ReturnsSuccess()
    {
        // Arrange
        var age1 = Age.Create(25).Value;
        var age2 = Age.Create(30).Value;

        // Act
        var result = AgeOperations.Add(age1, age2);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Value.Should().Be(55);
    }

    [TestMethod]
    public void Add_ExceedingMax_ReturnsFailure()
    {
        // Arrange
        var age1 = Age.Create(100).Value;
        var age2 = Age.Create(60).Value;

        // Act
        var result = AgeOperations.Add(age1, age2);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("exceed");
    }
}
```

## Why It's Important

1. **Type Safety**: Invalid states impossible to create
2. **Self-Documenting**: Types encode constraints
3. **Early Validation**: Errors caught at construction
4. **Invariants**: Values always valid once created
5. **Correctness**: Compiler enforces validation

## Guidelines

**Testing Refinement Types**
- Test valid values within constraints
- Test boundary values (min, max)
- Test invalid values (below min, above max)
- Test all validation rules
- Test composition with other types

**Validation Testing**
- Test each constraint independently
- Test error messages are clear
- Test normalization (case, whitespace)
- Test edge cases

**Common Patterns**
- `.IsSuccess.Should().BeTrue()`
- `.IsFailure.Should().BeTrue()`
- `.Error.Should().Contain("text")`
- Test boundary conditions
- Test all failure paths

## Common Pitfalls

❌ **Not testing boundaries**
```csharp
// Incomplete - missing 0 and 150
Age.Create(25).IsSuccess.Should().BeTrue();
Age.Create(-1).IsFailure.Should().BeTrue();
```

✅ **Test all boundaries**
```csharp
// Complete
Age.Create(0).IsSuccess.Should().BeTrue();
Age.Create(150).IsSuccess.Should().BeTrue();
Age.Create(-1).IsFailure.Should().BeTrue();
Age.Create(151).IsFailure.Should().BeTrue();
```

❌ **Not testing all validation rules**
```csharp
// Incomplete - only tests one rule
EmailAddress.Create("").IsFailure.Should().BeTrue();
```

✅ **Test each rule**
```csharp
// Complete
EmailAddress.Create("").IsFailure.Should().BeTrue();
EmailAddress.Create("no-at").IsFailure.Should().BeTrue();
EmailAddress.Create("no@dot").IsFailure.Should().BeTrue();
```

## See Also

- [Testing Smart Constructors](./testing-smart-constructors.md) — validated construction
- [Testing Domain Invariants](./testing-domain-invariants.md) — business rules
- [Refinement Types](../patterns/refinement-types.md) — refinement pattern
- [Smart Constructors](../patterns/smart-constructors.md) — construction validation
