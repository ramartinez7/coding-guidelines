# One Assertion Per Test

> Each test should verify a single logical concept—multiple physical assertions are fine if they verify the same behavior.

## Problem

Tests that verify multiple unrelated behaviors become hard to understand, fail with confusing messages, and violate the Single Responsibility Principle. When one assertion fails, subsequent assertions don't run, hiding additional issues.

## Guideline

**One Logical Concept, Not One Assert Statement**

The rule is not about counting `Assert` calls—it's about testing one behavior or outcome. Multiple assertions are fine if they all verify the same logical concept.

## Examples

### ❌ Before - Multiple Concepts

```csharp
[Fact]
public void TestOrder()
{
    // Testing creation AND validation AND processing
    var order = Order.Create(CustomerId.New(), Money.USD(100m));
    Assert.NotNull(order);
    Assert.Equal(OrderStatus.Pending, order.Status);
    
    var validationResult = order.Validate();
    Assert.True(validationResult.IsSuccess);
    
    var processResult = order.Process();
    Assert.True(processResult.IsSuccess);
    Assert.Equal(OrderStatus.Processed, order.Status);
}
```

### ✅ After - One Concept Per Test

```csharp
[Fact]
public void CreateOrder_WithValidInputs_CreatesOrderInPendingState()
{
    // Arrange
    var customerId = CustomerId.New();
    var amount = Money.USD(100m);

    // Act
    var order = Order.Create(customerId, amount);

    // Assert - All assertions verify creation behavior
    Assert.NotNull(order);
    Assert.Equal(OrderStatus.Pending, order.Status);
    Assert.Equal(customerId, order.CustomerId);
    Assert.Equal(amount, order.Total);
}

[Fact]
public void ValidateOrder_WithValidOrder_ReturnsSuccess()
{
    // Arrange
    var order = Order.Create(CustomerId.New(), Money.USD(100m));

    // Act
    var result = order.Validate();

    // Assert
    Assert.True(result.IsSuccess);
}

[Fact]
public void ProcessOrder_WithValidOrder_UpdatesStatusToProcessed()
{
    // Arrange
    var order = Order.Create(CustomerId.New(), Money.USD(100m));

    // Act
    var result = order.Process();

    // Assert - All assertions verify processing behavior
    Assert.True(result.IsSuccess);
    Assert.Equal(OrderStatus.Processed, order.Status);
}
```

### ✅ Multiple Assertions for Same Concept

```csharp
[Fact]
public void CreateEmailAddress_WithValidInput_NormalizesValue()
{
    // Arrange
    var input = "  USER@EXAMPLE.COM  ";

    // Act
    var result = EmailAddress.Create(input);

    // Assert - All verify normalization behavior
    Assert.True(result.IsSuccess);
    Assert.Equal("user@example.com", result.Value.ToString());
    Assert.DoesNotContain(" ", result.Value.ToString());
}

[Fact]
public void CreateMoney_WithAmount_StoresCorrectValues()
{
    // Arrange
    var amount = 99.99m;
    var currency = Currency.USD;

    // Act
    var money = Money.Create(amount, currency);

    // Assert - All verify the value object's state
    Assert.Equal(amount, money.Amount);
    Assert.Equal(currency, money.Currency);
    Assert.Equal("$99.99", money.ToString());
}
```

## When Multiple Assertions Are Acceptable

**Testing value object properties**
```csharp
var address = Address.Create("123 Main St", "Seattle", "WA", "98101");
Assert.Equal("123 Main St", address.Street);
Assert.Equal("Seattle", address.City);
Assert.Equal("WA", address.State);
Assert.Equal("98101", address.ZipCode);
```

**Testing collection contents**
```csharp
var items = order.LineItems;
Assert.Equal(3, items.Count);
Assert.Contains(items, item => item.ProductId == productId1);
Assert.Contains(items, item => item.ProductId == productId2);
```

**Testing complex objects**
```csharp
var result = parser.Parse(json);
Assert.True(result.IsSuccess);
Assert.NotNull(result.Value);
Assert.Equal(expectedId, result.Value.Id);
Assert.Equal(expectedName, result.Value.Name);
```

## When to Split Tests

**Different error conditions**
```csharp
// Separate tests
CreateEmailAddress_WithNullInput_ReturnsFailure()
CreateEmailAddress_WithEmptyString_ReturnsFailure()
CreateEmailAddress_WithNoAtSign_ReturnsFailure()
```

**Different behaviors**
```csharp
// Separate tests
ProcessOrder_WithValidOrder_UpdatesStatus()
ProcessOrder_WithValidOrder_SendsConfirmationEmail()
ProcessOrder_WithValidOrder_RecordsAuditLog()
```

**Different scenarios**
```csharp
// Separate tests
CalculateDiscount_ForStandardCustomer_AppliesStandardRate()
CalculateDiscount_ForPremiumCustomer_AppliesPremiumRate()
CalculateDiscount_ForVIPCustomer_AppliesVIPRate()
```

## Benefits

1. **Clear failure messages**: Know exactly which behavior failed
2. **Complete feedback**: All assertions run even if earlier ones fail (in different tests)
3. **Better documentation**: Test names describe specific behaviors
4. **Easier maintenance**: Modify tests for one behavior without affecting others
5. **Focused tests**: Each test has a clear, single purpose

## Anti-Pattern: Testing Everything

```csharp
// ❌ Don't do this
[Fact]
public void TestEntireUserWorkflow()
{
    var user = CreateUser();
    var loginResult = Login(user);
    var updateResult = UpdateProfile(user);
    var order = CreateOrder(user);
    var paymentResult = ProcessPayment(order);
    var shipmentResult = ShipOrder(order);
    // ... this is an integration test, not a unit test
}
```

## See Also

- [Arrange-Act-Assert Pattern](./arrange-act-assert.md) — test structure
- [Test Naming Conventions](./test-naming-conventions.md) — communicating test purpose
- [Test Isolation and Independence](./test-isolation-independence.md) — independent tests
- [Parameterized Tests](./parameterized-tests.md) — testing multiple scenarios
