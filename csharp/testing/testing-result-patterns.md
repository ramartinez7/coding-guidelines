# Testing Result Patterns

> Verify Result<T> monadic error handling, success/failure cases, and composition using FluentAssertions.

## Problem

Result patterns provide explicit error handling without exceptions. Tests must verify both success and failure paths, error messages, and result composition.

## Pattern

Test Result<T> for success values, failure errors, mapping operations, and monadic composition using FluentAssertions.

## Example

### ❌ Before - Exception-Based Testing

```csharp
[TestMethod]
public void CreateCustomer_WithInvalidEmail_Throws()
{
    Assert.ThrowsException<ValidationException>(() =>
        Customer.Create("John", "invalid-email")
    );
    // ❌ Uses exceptions for flow control
}
```

### ✅ After - Result Pattern Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

public record Result<T>
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public T Value { get; }
    public string Error { get; }

    private Result(T value)
    {
        IsSuccess = true;
        Value = value;
        Error = string.Empty;
    }

    private Result(string error)
    {
        IsSuccess = false;
        Value = default!;
        Error = error;
    }

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(string error) => new(error);

    public Result<TNew> Map<TNew>(Func<T, TNew> mapper) =>
        IsSuccess
            ? Result<TNew>.Success(mapper(Value))
            : Result<TNew>.Failure(Error);

    public Result<TNew> Bind<TNew>(Func<T, Result<TNew>> binder) =>
        IsSuccess
            ? binder(Value)
            : Result<TNew>.Failure(Error);
}

[TestClass]
public class ResultPatternTests
{
    [TestMethod]
    public void Success_CreatesSuccessfulResult()
    {
        // Act
        var result = Result<int>.Success(42);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.IsFailure.Should().BeFalse();
        result.Value.Should().Be(42);
        result.Error.Should().BeEmpty();
    }

    [TestMethod]
    public void Failure_CreatesFailedResult()
    {
        // Act
        var result = Result<int>.Failure("Something went wrong");

        // Assert
        result.IsFailure.Should().BeTrue();
        result.IsSuccess.Should().BeFalse();
        result.Error.Should().Be("Something went wrong");
    }
}
```

## Testing Result Creation

```csharp
public class Email
{
    public string Value { get; }

    private Email(string value) => Value = value;

    public static Result<Email> Create(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return Result<Email>.Failure("Email cannot be empty");

        if (!value.Contains("@"))
            return Result<Email>.Failure("Email must contain @");

        if (value.Length > 254)
            return Result<Email>.Failure("Email too long");

        return Result<Email>.Success(new Email(value));
    }
}

[TestClass]
public class EmailResultTests
{
    [TestMethod]
    public void Create_WithValidEmail_ReturnsSuccess()
    {
        // Act
        var result = Email.Create("user@example.com");

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeNull();
        result.Value.Value.Should().Be("user@example.com");
    }

    [TestMethod]
    public void Create_WithEmptyEmail_ReturnsFailure()
    {
        // Act
        var result = Email.Create("");

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("empty");
    }

    [TestMethod]
    public void Create_WithoutAtSign_ReturnsFailure()
    {
        // Act
        var result = Email.Create("invalid-email");

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("@");
    }

    [TestMethod]
    public void Create_WithTooLongEmail_ReturnsFailure()
    {
        // Arrange
        var longEmail = new string('a', 255) + "@example.com";

        // Act
        var result = Email.Create(longEmail);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("too long");
    }
}
```

## Testing Result Mapping

```csharp
[TestClass]
public class ResultMappingTests
{
    [TestMethod]
    public void Map_OnSuccess_TransformsValue()
    {
        // Arrange
        var result = Result<int>.Success(42);

        // Act
        var mapped = result.Map(x => x.ToString());

        // Assert
        mapped.IsSuccess.Should().BeTrue();
        mapped.Value.Should().Be("42");
    }

    [TestMethod]
    public void Map_OnFailure_PropagatesError()
    {
        // Arrange
        var result = Result<int>.Failure("Error occurred");

        // Act
        var mapped = result.Map(x => x.ToString());

        // Assert
        mapped.IsFailure.Should().BeTrue();
        mapped.Error.Should().Be("Error occurred");
    }

    [TestMethod]
    public void Map_ChainedMappings_TransformsSuccessfully()
    {
        // Arrange
        var result = Result<int>.Success(10);

        // Act
        var final = result
            .Map(x => x * 2)
            .Map(x => x + 5)
            .Map(x => x.ToString());

        // Assert
        final.IsSuccess.Should().BeTrue();
        final.Value.Should().Be("25");
    }
}
```

## Testing Result Binding (Bind/FlatMap)

```csharp
[TestClass]
public class ResultBindingTests
{
    [TestMethod]
    public void Bind_OnSuccess_ExecutesBinder()
    {
        // Arrange
        var result = Result<string>.Success("user@example.com");

        // Act
        var bound = result.Bind(value => Email.Create(value));

        // Assert
        bound.IsSuccess.Should().BeTrue();
        bound.Value.Value.Should().Be("user@example.com");
    }

    [TestMethod]
    public void Bind_OnFailure_PropagatesError()
    {
        // Arrange
        var result = Result<string>.Failure("Initial error");

        // Act
        var bound = result.Bind(value => Email.Create(value));

        // Assert
        bound.IsFailure.Should().BeTrue();
        bound.Error.Should().Be("Initial error");
    }

    [TestMethod]
    public void Bind_WithFailingBinder_ReturnsBinderFailure()
    {
        // Arrange
        var result = Result<string>.Success("invalid-email");

        // Act
        var bound = result.Bind(value => Email.Create(value));

        // Assert
        bound.IsFailure.Should().BeTrue();
        bound.Error.Should().Contain("@");
    }

    [TestMethod]
    public void Bind_ChainedBindings_ComposesCorrectly()
    {
        // Arrange
        var emailResult = Email.Create("user@example.com");

        // Act
        var final = emailResult
            .Bind(email => ValidateDomain(email))
            .Bind(email => CheckBlacklist(email));

        // Assert - Depends on implementation
        final.IsSuccess.Should().BeTrue();
    }

    private Result<Email> ValidateDomain(Email email) =>
        email.Value.EndsWith("@example.com")
            ? Result<Email>.Success(email)
            : Result<Email>.Failure("Invalid domain");

    private Result<Email> CheckBlacklist(Email email) =>
        Result<Email>.Success(email); // Simplified
}
```

## Testing Result with Async Operations

```csharp
public static class AsyncResultExtensions
{
    public static async Task<Result<TNew>> MapAsync<T, TNew>(
        this Result<T> result,
        Func<T, Task<TNew>> mapper)
    {
        if (result.IsFailure)
            return Result<TNew>.Failure(result.Error);

        var value = await mapper(result.Value);
        return Result<TNew>.Success(value);
    }

    public static async Task<Result<TNew>> BindAsync<T, TNew>(
        this Result<T> result,
        Func<T, Task<Result<TNew>>> binder)
    {
        return result.IsFailure
            ? Result<TNew>.Failure(result.Error)
            : await binder(result.Value);
    }
}

[TestClass]
public class AsyncResultTests
{
    [TestMethod]
    public async Task MapAsync_OnSuccess_TransformsAsync()
    {
        // Arrange
        var result = Result<int>.Success(42);

        // Act
        var mapped = await result.MapAsync(async x =>
        {
            await Task.Delay(1);
            return x.ToString();
        });

        // Assert
        mapped.IsSuccess.Should().BeTrue();
        mapped.Value.Should().Be("42");
    }

    [TestMethod]
    public async Task BindAsync_OnSuccess_ExecutesBinderAsync()
    {
        // Arrange
        var result = Result<string>.Success("user@example.com");

        // Act
        var bound = await result.BindAsync(async value =>
        {
            await Task.Delay(1);
            return Email.Create(value);
        });

        // Assert
        bound.IsSuccess.Should().BeTrue();
    }

    [TestMethod]
    public async Task BindAsync_OnFailure_PropagatesError()
    {
        // Arrange
        var result = Result<string>.Failure("Error");

        // Act
        var bound = await result.BindAsync(async value =>
        {
            await Task.Delay(1);
            return Email.Create(value);
        });

        // Assert
        bound.IsFailure.Should().BeTrue();
        bound.Error.Should().Be("Error");
    }
}
```

## Testing Result Combination

```csharp
public static class ResultCombinators
{
    public static Result<(T1, T2)> Combine<T1, T2>(
        Result<T1> result1,
        Result<T2> result2)
    {
        if (result1.IsFailure)
            return Result<(T1, T2)>.Failure(result1.Error);

        if (result2.IsFailure)
            return Result<(T1, T2)>.Failure(result2.Error);

        return Result<(T1, T2)>.Success((result1.Value, result2.Value));
    }
}

[TestClass]
public class ResultCombinationTests
{
    [TestMethod]
    public void Combine_BothSuccess_ReturnsSuccessWithTuple()
    {
        // Arrange
        var result1 = Result<int>.Success(42);
        var result2 = Result<string>.Success("hello");

        // Act
        var combined = ResultCombinators.Combine(result1, result2);

        // Assert
        combined.IsSuccess.Should().BeTrue();
        combined.Value.Item1.Should().Be(42);
        combined.Value.Item2.Should().Be("hello");
    }

    [TestMethod]
    public void Combine_FirstFailure_ReturnsFirstError()
    {
        // Arrange
        var result1 = Result<int>.Failure("Error 1");
        var result2 = Result<string>.Success("hello");

        // Act
        var combined = ResultCombinators.Combine(result1, result2);

        // Assert
        combined.IsFailure.Should().BeTrue();
        combined.Error.Should().Be("Error 1");
    }

    [TestMethod]
    public void Combine_SecondFailure_ReturnsSecondError()
    {
        // Arrange
        var result1 = Result<int>.Success(42);
        var result2 = Result<string>.Failure("Error 2");

        // Act
        var combined = ResultCombinators.Combine(result1, result2);

        // Assert
        combined.IsFailure.Should().BeTrue();
        combined.Error.Should().Be("Error 2");
    }

    [TestMethod]
    public void Combine_BothFailure_ReturnsFirstError()
    {
        // Arrange
        var result1 = Result<int>.Failure("Error 1");
        var result2 = Result<string>.Failure("Error 2");

        // Act
        var combined = ResultCombinators.Combine(result1, result2);

        // Assert
        combined.IsFailure.Should().BeTrue();
        combined.Error.Should().Be("Error 1");
    }
}
```

## Testing Result Match

```csharp
public static class ResultExtensions
{
    public static TResult Match<T, TResult>(
        this Result<T> result,
        Func<T, TResult> onSuccess,
        Func<string, TResult> onFailure)
    {
        return result.IsSuccess
            ? onSuccess(result.Value)
            : onFailure(result.Error);
    }
}

[TestClass]
public class ResultMatchTests
{
    [TestMethod]
    public void Match_OnSuccess_ExecutesSuccessFunction()
    {
        // Arrange
        var result = Result<int>.Success(42);

        // Act
        var output = result.Match(
            onSuccess: value => $"Success: {value}",
            onFailure: error => $"Failure: {error}"
        );

        // Assert
        output.Should().Be("Success: 42");
    }

    [TestMethod]
    public void Match_OnFailure_ExecutesFailureFunction()
    {
        // Arrange
        var result = Result<int>.Failure("Something went wrong");

        // Act
        var output = result.Match(
            onSuccess: value => $"Success: {value}",
            onFailure: error => $"Failure: {error}"
        );

        // Assert
        output.Should().Be("Failure: Something went wrong");
    }
}
```

## Why It's Important

1. **Explicit Errors**: Make error handling explicit in types
2. **No Exceptions**: Avoid exceptions for expected errors
3. **Composability**: Chain operations functionally
4. **Type Safety**: Compiler enforces error handling
5. **Railway-Oriented**: Clear success/failure paths

## Guidelines

**Testing Results**
- Test success path with valid values
- Test failure path with error messages
- Test mapping and binding operations
- Test async result operations
- Test result combinations

**Common Patterns**
- `IsSuccess.Should().BeTrue()`
- `IsFailure.Should().BeTrue()`
- `Value.Should().Be(expected)`
- `Error.Should().Contain("text")`
- Test chained operations

**Result Assertions**
- Verify success/failure state
- Check value on success
- Check error message on failure
- Test transformation correctness

## Common Pitfalls

❌ **Accessing Value on failure**
```csharp
// Dangerous
var result = Email.Create("invalid");
var value = result.Value; // Undefined behavior!
```

✅ **Check success first**
```csharp
// Safe
var result = Email.Create("invalid");
if (result.IsSuccess)
{
    var value = result.Value;
}
```

❌ **Not testing failure cases**
```csharp
// Incomplete
var result = Email.Create("valid@example.com");
result.IsSuccess.Should().BeTrue();
```

✅ **Test both paths**
```csharp
// Complete
Email.Create("valid@example.com").IsSuccess.Should().BeTrue();
Email.Create("invalid").IsFailure.Should().BeTrue();
Email.Create("").IsFailure.Should().BeTrue();
```

## See Also

- [Testing Option Patterns](./testing-option-patterns.md) — optional values
- [Testing Exceptions](./testing-exceptions.md) — exception testing
- [Result Monad](../patterns/result-monad.md) — result pattern
- [Honest Functions](../patterns/honest-functions.md) — explicit errors
