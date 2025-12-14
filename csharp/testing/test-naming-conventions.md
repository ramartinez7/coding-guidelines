# Test Naming Conventions

> Name tests to reveal their purpose—what is being tested, under what conditions, and what is expected.

## Problem

Generic test names like `Test1()` or `ValidateUser()` don't communicate what they're testing. When a test fails, unclear names force developers to read the entire test to understand what broke.

## Pattern

Use naming patterns that make test intent self-documenting:

**Pattern 1: MethodName_Scenario_ExpectedBehavior**

```csharp
CreateOrder_WithInvalidCustomerId_ReturnsValidationError()
ProcessPayment_WhenBalanceInsufficient_ThrowsInsufficientFundsException()
```

**Pattern 2: Should_ExpectedBehavior_When_Scenario**

```csharp
Should_ReturnValidationError_When_CustomerIdIsInvalid()
Should_ThrowInsufficientFundsException_When_BalanceIsInsufficient()
```

**Pattern 3: Given_When_Then (BDD style)**

```csharp
Given_InvalidCustomerId_When_CreatingOrder_Then_ReturnsValidationError()
Given_InsufficientBalance_When_ProcessingPayment_Then_ThrowsException()
```

## Examples

### ❌ Before

```csharp
[Fact]
public void Test1()
{
    var email = EmailAddress.Create("invalid");
    Assert.True(email.IsFailure);
}

[Fact]
public void TestEmail()
{
    var email = EmailAddress.Create("user@example.com");
    Assert.True(email.IsSuccess);
}

[Fact]
public void CreateOrder()
{
    var order = Order.Create(CustomerId.New(), Money.USD(100m));
    Assert.NotNull(order);
}
```

### ✅ After

```csharp
[Fact]
public void CreateEmailAddress_WithInvalidFormat_ReturnsFailure()
{
    // Arrange
    var invalidEmail = "invalid";

    // Act
    var result = EmailAddress.Create(invalidEmail);

    // Assert
    Assert.True(result.IsFailure);
    Assert.Contains("email format", result.Error, StringComparison.OrdinalIgnoreCase);
}

[Fact]
public void CreateEmailAddress_WithValidFormat_ReturnsSuccess()
{
    // Arrange
    var validEmail = "user@example.com";

    // Act
    var result = EmailAddress.Create(validEmail);

    // Assert
    Assert.True(result.IsSuccess);
    Assert.Equal(validEmail, result.Value.ToString());
}

[Fact]
public void CreateOrder_WithValidParameters_CreatesOrderInPendingState()
{
    // Arrange
    var customerId = CustomerId.New();
    var amount = Money.USD(100m);

    // Act
    var order = Order.Create(customerId, amount);

    // Assert
    Assert.NotNull(order);
    Assert.Equal(OrderStatus.Pending, order.Status);
    Assert.Equal(customerId, order.CustomerId);
}
```

## Guidelines

**Choose a pattern and be consistent**
- Pick one naming pattern for your team
- Use it across the entire codebase
- Document it in team coding standards

**Be specific**
- State exact input conditions: `EmptyString`, `NullValue`, `NegativeAmount`
- State exact expected outcomes: `ThrowsArgumentException`, `ReturnsValidationError`
- Avoid vague terms like `Valid`, `Invalid`, `Works`, `Fails`

**Keep names readable**
- Use underscores to separate parts (improves readability)
- Avoid abbreviations unless universally understood
- Don't sacrifice clarity for brevity

**Test method names can be long**
- Unlike production code, long test names are acceptable
- Clarity is more important than brevity
- 50-80 characters is normal for descriptive test names

## Pattern Comparison

| Pattern | Pros | Cons |
|---------|------|------|
| MethodName_Scenario_Expected | Most common in C#, clear structure | Can be verbose |
| Should_Expected_When_Scenario | Reads like specification | Less common in C# |
| Given_When_Then | BDD-style, very explicit | Longest names |

## See Also

- [Arrange-Act-Assert Pattern](./arrange-act-assert.md) — test structure
- [One Assertion Per Test](./one-assertion-per-test.md) — focused tests
- [Testing Domain Invariants](./testing-domain-invariants.md) — naming tests for business rules
