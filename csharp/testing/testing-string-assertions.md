# Testing String Assertions

> Verify string content, format, and patterns using FluentAssertions' comprehensive string testing capabilities.

## Problem

String testing often involves multiple aspects: exact content, substrings, patterns, formatting, and case sensitivity. Writing custom assertion logic for each scenario is verbose and error-prone.

## Pattern

Use FluentAssertions' string assertions to test content, format, whitespace, patterns, and case variations expressively.

## Example

### ❌ Before - Verbose String Testing

```csharp
[TestMethod]
public void FormatCustomerName_ReturnsFormattedString()
{
    var result = formatter.Format("John", "Doe");
    Assert.IsTrue(result.Contains("John"));
    Assert.IsTrue(result.Contains("Doe"));
    Assert.IsTrue(result.StartsWith("Customer:"));
    // ❌ Verbose and unclear error messages
}
```

### ✅ After - Fluent String Assertions

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class CustomerNameFormatterTests
{
    [TestMethod]
    public void FormatCustomerName_ReturnsFormattedString()
    {
        // Arrange
        var formatter = new CustomerNameFormatter();

        // Act
        var result = formatter.Format("John", "Doe");

        // Assert
        result.Should().StartWith("Customer:");
        result.Should().Contain("John");
        result.Should().Contain("Doe");
        result.Should().EndWith(".");
    }

    [TestMethod]
    public void FormatCustomerName_WithMiddleName_IncludesAll()
    {
        // Arrange
        var formatter = new CustomerNameFormatter();

        // Act
        var result = formatter.Format("John", "Middle", "Doe");

        // Assert
        result.Should().Be("Customer: John Middle Doe.");
    }

    [TestMethod]
    public void FormatCustomerName_IgnoresCase_WhenSpecified()
    {
        // Arrange
        var formatter = new CustomerNameFormatter();

        // Act
        var result = formatter.Format("john", "doe");

        // Assert
        result.Should().ContainEquivalentOf("JOHN");
        result.Should().ContainEquivalentOf("DOE");
    }
}
```

## String Content Assertions

```csharp
[TestClass]
public class StringContentTests
{
    [TestMethod]
    public void String_HasExpectedContent()
    {
        // Arrange
        var message = "Hello World";

        // Act & Assert
        message.Should().Be("Hello World");
        message.Should().NotBe("Goodbye World");
    }

    [TestMethod]
    public void String_ContainsSubstring()
    {
        // Arrange
        var email = "user@example.com";

        // Act & Assert
        email.Should().Contain("@");
        email.Should().Contain("example");
        email.Should().NotContain("test");
    }

    [TestMethod]
    public void String_ContainsSubstring_IgnoringCase()
    {
        // Arrange
        var text = "Hello World";

        // Act & Assert
        text.Should().ContainEquivalentOf("HELLO");
        text.Should().ContainEquivalentOf("world");
    }

    [TestMethod]
    public void String_StartsAndEndsWithExpectedText()
    {
        // Arrange
        var path = "/api/customers/123";

        // Act & Assert
        path.Should().StartWith("/api");
        path.Should().EndWith("123");
        path.Should().NotStartWith("http");
    }

    [TestMethod]
    public void String_StartsAndEnds_IgnoringCase()
    {
        // Arrange
        var header = "Content-Type: application/json";

        // Act & Assert
        header.Should().StartWithEquivalentOf("CONTENT-TYPE");
        header.Should().EndWithEquivalentOf("JSON");
    }
}
```

## String Pattern and Format Testing

```csharp
[TestClass]
public class StringPatternTests
{
    [TestMethod]
    public void Email_MatchesExpectedPattern()
    {
        // Arrange
        var email = "user@example.com";

        // Act & Assert
        email.Should().MatchRegex(@"^[^@]+@[^@]+\.[^@]+$");
    }

    [TestMethod]
    public void PhoneNumber_MatchesExpectedFormat()
    {
        // Arrange
        var phone = "(555) 123-4567";

        // Act & Assert
        phone.Should().MatchRegex(@"^\(\d{3}\) \d{3}-\d{4}$");
    }

    [TestMethod]
    public void Guid_HasExpectedFormat()
    {
        // Arrange
        var guidString = Guid.NewGuid().ToString();

        // Act & Assert
        guidString.Should().MatchRegex(
            @"^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$"
        );
    }
}
```

## String Length and Empty Testing

```csharp
[TestClass]
public class StringLengthTests
{
    [TestMethod]
    public void String_HasExpectedLength()
    {
        // Arrange
        var code = "ABC123";

        // Act & Assert
        code.Should().HaveLength(6);
        code.Should().NotBeEmpty();
    }

    [TestMethod]
    public void String_MeetsLengthConstraints()
    {
        // Arrange
        var password = "MySecurePassword123!";

        // Act & Assert
        password.Should().HaveLength(20);
        password.Length.Should().BeGreaterThanOrEqualTo(8);
        password.Length.Should().BeLessThanOrEqualTo(50);
    }

    [TestMethod]
    public void String_IsEmpty()
    {
        // Arrange
        var empty = string.Empty;

        // Act & Assert
        empty.Should().BeEmpty();
        empty.Should().NotBeNull();
    }

    [TestMethod]
    public void String_IsNullOrEmpty()
    {
        // Arrange
        string? nullString = null;
        var emptyString = string.Empty;

        // Act & Assert
        nullString.Should().BeNullOrEmpty();
        emptyString.Should().BeNullOrEmpty();
    }

    [TestMethod]
    public void String_IsNullOrWhiteSpace()
    {
        // Arrange
        var whitespace = "   ";

        // Act & Assert
        whitespace.Should().BeNullOrWhiteSpace();
    }
}
```

## Whitespace and Formatting

```csharp
[TestClass]
public class StringWhitespaceTests
{
    [TestMethod]
    public void String_EqualsIgnoringWhitespace()
    {
        // Arrange
        var actual = "  Hello   World  ";
        var expected = "Hello World";

        // Act & Assert
        actual.Trim().Should().Be(expected);
    }

    [TestMethod]
    public void String_EqualsIgnoringLeadingWhitespace()
    {
        // Arrange
        var text = "  Hello";

        // Act & Assert
        text.Should().EndWith("Hello");
    }

    [TestMethod]
    public void MultilineString_HasExpectedLines()
    {
        // Arrange
        var multiline = "Line1\nLine2\nLine3";

        // Act & Assert
        var lines = multiline.Split('\n');
        lines.Should().HaveCount(3);
        lines[0].Should().Be("Line1");
        lines[1].Should().Be("Line2");
        lines[2].Should().Be("Line3");
    }
}
```

## String Equivalence and Case

```csharp
[TestClass]
public class StringCaseTests
{
    [TestMethod]
    public void String_EqualsIgnoringCase()
    {
        // Arrange
        var actual = "Hello World";
        var expected = "hello world";

        // Act & Assert
        actual.Should().BeEquivalentTo(expected);
    }

    [TestMethod]
    public void String_IsUpperCase()
    {
        // Arrange
        var code = "ABC123";

        // Act & Assert
        code.Should().Match(s => s.All(c => !char.IsLetter(c) || char.IsUpper(c)));
    }

    [TestMethod]
    public void String_IsLowerCase()
    {
        // Arrange
        var slug = "my-product-slug";

        // Act & Assert
        slug.Should().Match(s => s.All(c => !char.IsLetter(c) || char.IsLower(c)));
    }
}
```

## Domain-Specific String Testing

```csharp
public record Email
{
    public string Value { get; }

    private Email(string value) => Value = value;

    public static Result<Email> Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return Result.Failure<Email>("Email cannot be empty");

        if (!value.Contains("@"))
            return Result.Failure<Email>("Email must contain @");

        return Result.Success(new Email(value));
    }
}

[TestClass]
public class EmailStringTests
{
    [TestMethod]
    public void Email_Create_WithValidFormat_ReturnsSuccess()
    {
        // Arrange
        var emailString = "user@example.com";

        // Act
        var result = Email.Create(emailString);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Value.Should().Be(emailString);
        result.Value.Value.Should().Contain("@");
        result.Value.Value.Should().MatchRegex(@"^[^@]+@[^@]+$");
    }

    [TestMethod]
    public void Email_Create_WithInvalidFormat_ReturnsFailure()
    {
        // Arrange
        var emailString = "invalid-email";

        // Act
        var result = Email.Create(emailString);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("@");
    }
}
```

## Why It's Important

1. **Clear Intent**: Assertions express what you're testing
2. **Better Errors**: Failures show expected vs actual clearly
3. **Less Code**: Built-in assertions reduce boilerplate
4. **Maintainability**: Easy to update and understand
5. **Type Safety**: Compile-time checking

## Guidelines

**Common String Assertions**
- `.Should().Be(expected)` - exact match
- `.Should().Contain(substring)` - substring match
- `.Should().StartWith()` / `.EndWith()` - prefix/suffix
- `.Should().BeEquivalentTo()` - case-insensitive match
- `.Should().MatchRegex(pattern)` - pattern matching
- `.Should().BeEmpty()` / `.BeNullOrEmpty()` - empty checks
- `.Should().HaveLength(n)` - length verification

**Case Sensitivity**
- Default methods are case-sensitive
- Use `EquivalentOf` suffix for case-insensitive (e.g., `ContainEquivalentOf`)
- Use `.BeEquivalentTo()` for case-insensitive equality

## Common Pitfalls

❌ **Using Contains for exact match**
```csharp
// Imprecise - would match "Hello World!" too
result.Should().Contain("Hello");
```

✅ **Use exact match when appropriate**
```csharp
// Precise
result.Should().Be("Hello");
```

❌ **Not specifying case sensitivity**
```csharp
// Ambiguous
result.Should().Contain("hello");
```

✅ **Be explicit about case**
```csharp
// Clear intent
result.Should().ContainEquivalentOf("hello"); // Case-insensitive
result.Should().Contain("hello"); // Case-sensitive
```

## See Also

- [Testing Value Objects](./testing-value-objects.md) — string-based value objects
- [Testing Domain Invariants](./testing-domain-invariants.md) — string validation rules
- [Smart Constructors](../patterns/smart-constructors.md) — validated string types
- [Refinement Types](../patterns/refinement-types.md) — constrained strings
