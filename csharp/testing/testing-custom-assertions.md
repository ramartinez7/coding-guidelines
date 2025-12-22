# Custom Assertions

> Create domain-specific assertions for clearer, more maintainable tests—express business validations in the language of your domain.

## Problem

Generic assertions like `Assert.True()` and `Assert.Equal()` don't express domain intent. Tests become verbose when checking complex domain rules, and error messages aren't helpful.

## Pattern

Create custom assertion methods that encapsulate domain validation logic, provide clear error messages, and make tests read like specifications.

## Example

### ❌ Before - Generic Assertions

```csharp
[Fact]
public void CreateOrder_ValidOrder_HasCorrectState()
{
    // Arrange
    var customerId = CustomerId.New();
    var items = new[] { CreateOrderItem("PROD-001", 2, 10m) };

    // Act
    var order = Order.Create(customerId, items);

    // Assert
    Assert.NotNull(order);
    Assert.Equal(OrderStatus.Pending, order.Status);
    Assert.Equal(customerId, order.CustomerId);
    Assert.Equal(1, order.Items.Count);
    Assert.Equal(20m, order.Total.Amount);
    Assert.Equal("USD", order.Total.Currency);
    Assert.True(order.CreatedAt <= DateTime.UtcNow);
    Assert.True(order.CreatedAt >= DateTime.UtcNow.AddSeconds(-5));
}
```

### ✅ After - Custom Assertions

```csharp
[Fact]
public void CreateOrder_ValidOrder_HasCorrectState()
{
    // Arrange
    var customerId = CustomerId.New();
    var items = new[] { CreateOrderItem("PROD-001", 2, 10m) };

    // Act
    var order = Order.Create(customerId, items);

    // Assert
    order.Should().BePendingOrder()
        .WithCustomer(customerId)
        .WithItemCount(1)
        .WithTotal(Money.USD(20m))
        .WithRecentCreationTime();
}
```

## Creating Custom Assertions

### Basic Custom Assertion Class

```csharp
public class OrderAssertions
{
    private readonly Order subject;

    public OrderAssertions(Order subject)
    {
        this.subject = subject;
    }

    public OrderAssertions BePendingOrder(string because = "")
    {
        subject.Should().NotBeNull(because);
        subject.Status.Should().Be(OrderStatus.Pending, because);
        return this;
    }

    public OrderAssertions WithCustomer(CustomerId customerId, string because = "")
    {
        subject.CustomerId.Should().Be(customerId, because);
        return this;
    }

    public OrderAssertions WithItemCount(int expectedCount, string because = "")
    {
        subject.Items.Should().HaveCount(expectedCount, because);
        return this;
    }

    public OrderAssertions WithTotal(Money expectedTotal, string because = "")
    {
        subject.Total.Should().Be(expectedTotal, because);
        return this;
    }

    public OrderAssertions WithRecentCreationTime(int maxSecondsAgo = 5, string because = "")
    {
        var now = DateTime.UtcNow;
        subject.CreatedAt.Should().BeOnOrBefore(now, because);
        subject.CreatedAt.Should().BeAfter(now.AddSeconds(-maxSecondsAgo), because);
        return this;
    }
}
```

### Extension Method for Fluent Syntax

```csharp
public static class OrderAssertionExtensions
{
    public static OrderAssertions Should(this Order order)
    {
        return new OrderAssertions(order);
    }
}
```

## FluentAssertions Custom Assertions

### Extending FluentAssertions

```csharp
using FluentAssertions;
using FluentAssertions.Execution;
using FluentAssertions.Primitives;

public class MoneyAssertions : ReferenceTypeAssertions<Money, MoneyAssertions>
{
    public MoneyAssertions(Money subject) : base(subject)
    {
    }

    protected override string Identifier => "money";

    public AndConstraint<MoneyAssertions> BePositive(string because = "", params object[] becauseArgs)
    {
        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .ForCondition(Subject.Amount > 0)
            .FailWith("Expected {context:money} to be positive{reason}, but found {0}.", Subject.Amount);

        return new AndConstraint<MoneyAssertions>(this);
    }

    public AndConstraint<MoneyAssertions> BeNegative(string because = "", params object[] becauseArgs)
    {
        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .ForCondition(Subject.Amount < 0)
            .FailWith("Expected {context:money} to be negative{reason}, but found {0}.", Subject.Amount);

        return new AndConstraint<MoneyAssertions>(this);
    }

    public AndConstraint<MoneyAssertions> BeInCurrency(string currency, string because = "", params object[] becauseArgs)
    {
        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .ForCondition(Subject.Currency == currency)
            .FailWith("Expected {context:money} to be in currency {0}{reason}, but found {1}.", 
                currency, Subject.Currency);

        return new AndConstraint<MoneyAssertions>(this);
    }

    public AndConstraint<MoneyAssertions> HaveAmount(decimal expected, string because = "", params object[] becauseArgs)
    {
        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .ForCondition(Subject.Amount == expected)
            .FailWith("Expected {context:money} to have amount {0}{reason}, but found {1}.", 
                expected, Subject.Amount);

        return new AndConstraint<MoneyAssertions>(this);
    }
}

public static class MoneyAssertionExtensions
{
    public static MoneyAssertions Should(this Money money)
    {
        return new MoneyAssertions(money);
    }
}
```

### Using Custom FluentAssertions

```csharp
[Fact]
public void Deposit_PositiveAmount_IncreasesBalance()
{
    // Arrange
    var account = BankAccount.Create(Money.USD(100m));
    var deposit = Money.USD(50m);

    // Act
    account.Deposit(deposit);

    // Assert
    account.Balance.Should()
        .BePositive()
        .And.BeInCurrency("USD")
        .And.HaveAmount(150m);
}
```

## Complex Domain Assertions

### Aggregate Root Assertions

```csharp
public class OrderAssertions : ReferenceTypeAssertions<Order, OrderAssertions>
{
    public OrderAssertions(Order subject) : base(subject)
    {
    }

    protected override string Identifier => "order";

    public AndConstraint<OrderAssertions> BeInState(
        OrderStatus expectedStatus, 
        string because = "", 
        params object[] becauseArgs)
    {
        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .ForCondition(Subject.Status == expectedStatus)
            .FailWith("Expected {context:order} to be in state {0}{reason}, but found {1}.",
                expectedStatus, Subject.Status);

        return new AndConstraint<OrderAssertions>(this);
    }

    public AndConstraint<OrderAssertions> ContainItem(
        string productId,
        int quantity,
        string because = "",
        params object[] becauseArgs)
    {
        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .ForCondition(Subject.Items.Any(i => 
                i.ProductId == productId && i.Quantity == quantity))
            .FailWith("Expected {context:order} to contain item {0} with quantity {1}{reason}.",
                productId, quantity);

        return new AndConstraint<OrderAssertions>(this);
    }

    public AndConstraint<OrderAssertions> HaveValidTotal(
        string because = "",
        params object[] becauseArgs)
    {
        var calculatedTotal = Subject.Items.Sum(i => i.Subtotal.Amount);

        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .ForCondition(Subject.Total.Amount == calculatedTotal)
            .FailWith("Expected {context:order} total {0} to equal sum of items {1}{reason}.",
                Subject.Total.Amount, calculatedTotal);

        return new AndConstraint<OrderAssertions>(this);
    }

    public AndConstraint<OrderAssertions> BeProcessable(
        string because = "",
        params object[] becauseArgs)
    {
        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .Given(() => Subject)
            .ForCondition(order => order.Status == OrderStatus.Pending)
            .FailWith("Expected {context:order} to be in Pending status for processing{reason}, but found {0}.",
                Subject.Status)
            .Then
            .ForCondition(order => order.Items.Any())
            .FailWith("Expected {context:order} to have at least one item for processing{reason}.")
            .Then
            .ForCondition(order => order.Total.Amount > 0)
            .FailWith("Expected {context:order} to have positive total for processing{reason}.");

        return new AndConstraint<OrderAssertions>(this);
    }
}
```

### Testing with Complex Assertions

```csharp
[Fact]
public void ProcessOrder_ValidOrder_TransitionsToProcessed()
{
    // Arrange
    var order = CreateValidOrder();

    // Act
    order.Process();

    // Assert
    order.Should()
        .BeInState(OrderStatus.Processed)
        .And.HaveValidTotal()
        .And.Subject.ProcessedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(1));
}

[Fact]
public void AddItem_ToOrder_UpdatesTotalCorrectly()
{
    // Arrange
    var order = CreateValidOrder();
    var newItem = OrderLineItem.Create("PROD-002", 1, Money.USD(15m));

    // Act
    order.AddItem(newItem);

    // Assert
    order.Should()
        .ContainItem("PROD-002", 1)
        .And.HaveValidTotal();
}
```

## Result Type Assertions

```csharp
public class ResultAssertions<T> : ReferenceTypeAssertions<Result<T>, ResultAssertions<T>>
{
    public ResultAssertions(Result<T> subject) : base(subject)
    {
    }

    protected override string Identifier => "result";

    public AndWhichConstraint<ResultAssertions<T>, T> BeSuccess(
        string because = "",
        params object[] becauseArgs)
    {
        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .ForCondition(Subject.IsSuccess)
            .FailWith("Expected {context:result} to be success{reason}, but it was a failure with error: {0}",
                Subject.IsFailure ? Subject.Error : "N/A");

        return new AndWhichConstraint<ResultAssertions<T>, T>(this, Subject.Value);
    }

    public AndConstraint<ResultAssertions<T>> BeFailure(
        string because = "",
        params object[] becauseArgs)
    {
        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .ForCondition(Subject.IsFailure)
            .FailWith("Expected {context:result} to be failure{reason}, but it was a success.");

        return new AndConstraint<ResultAssertions<T>>(this);
    }

    public AndConstraint<ResultAssertions<T>> HaveError(
        string expectedError,
        string because = "",
        params object[] becauseArgs)
    {
        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .ForCondition(Subject.IsFailure)
            .FailWith("Expected {context:result} to have error{reason}, but it was a success.")
            .Then
            .ForCondition(Subject.Error.Contains(expectedError))
            .FailWith("Expected {context:result} error to contain {0}{reason}, but found {1}.",
                expectedError, Subject.Error);

        return new AndConstraint<ResultAssertions<T>>(this);
    }
}

public static class ResultAssertionExtensions
{
    public static ResultAssertions<T> Should<T>(this Result<T> result)
    {
        return new ResultAssertions<T>(result);
    }
}
```

### Using Result Assertions

```csharp
[Fact]
public void CreateCustomer_ValidData_ReturnsSuccess()
{
    // Arrange
    var email = "customer@example.com";
    var name = "John Doe";

    // Act
    var result = Customer.Create(email, name);

    // Assert
    result.Should()
        .BeSuccess()
        .Which.Email.Should().Be(email);
}

[Fact]
public void CreateCustomer_InvalidEmail_ReturnsFailure()
{
    // Arrange
    var email = "invalid";
    var name = "John Doe";

    // Act
    var result = Customer.Create(email, name);

    // Assert
    result.Should()
        .BeFailure()
        .And.HaveError("email format");
}
```

## Collection Assertions

```csharp
public static class CollectionAssertionExtensions
{
    public static AndConstraint<TAssertions> ContainEquivalentItemsAs<TAssertions, T>(
        this SelfReferencingCollectionAssertions<T, TAssertions> assertions,
        IEnumerable<T> expected,
        string because = "",
        params object[] becauseArgs)
        where TAssertions : SelfReferencingCollectionAssertions<T, TAssertions>
    {
        return assertions.BeEquivalentTo(expected, because, becauseArgs);
    }

    public static AndConstraint<TAssertions> ContainItemMatching<TAssertions, T>(
        this SelfReferencingCollectionAssertions<T, TAssertions> assertions,
        Func<T, bool> predicate,
        string because = "",
        params object[] becauseArgs)
        where TAssertions : SelfReferencingCollectionAssertions<T, TAssertions>
    {
        return assertions.Contain(predicate, because, becauseArgs);
    }

    public static AndConstraint<TAssertions> AllMatchDomainRules<TAssertions, T>(
        this SelfReferencingCollectionAssertions<T, TAssertions> assertions,
        string because = "",
        params object[] becauseArgs)
        where TAssertions : SelfReferencingCollectionAssertions<T, TAssertions>
        where T : IValidatable
    {
        Execute.Assertion
            .BecauseOf(because, becauseArgs)
            .ForCondition(assertions.Subject.All(item => item.IsValid()))
            .FailWith("Expected all items to match domain rules{reason}.");

        return new AndConstraint<TAssertions>((TAssertions)assertions);
    }
}
```

## Guidelines

1. **Express domain concepts**: Assertions should speak the ubiquitous language
2. **Chain assertions**: Return `this` or `AndConstraint` for fluent chaining
3. **Clear error messages**: Provide context in failure messages
4. **Reusable**: Create assertions for frequently checked conditions
5. **Composable**: Build complex assertions from simpler ones

## Benefits

1. **Readability**: Tests read like specifications
2. **Maintainability**: Change validation logic in one place
3. **Better errors**: Domain-specific failure messages
4. **Discoverability**: IntelliSense shows domain operations
5. **Reusability**: Share assertions across test files

## See Also

- [Testing Domain Invariants](./testing-domain-invariants.md) — verifying business rules
- [Testing Result Patterns](./testing-result-patterns.md) — asserting on Result types
- [Testing Value Objects](./testing-value-objects.md) — value semantics assertions
- [Test Data Builders](./test-data-builders.md) — building test objects
