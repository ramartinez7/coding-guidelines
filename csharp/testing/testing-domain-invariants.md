# Testing Domain Invariants

> Verify business rules are enforced—test that invalid states cannot be created or reached.

## Problem

Domain invariants are business rules that must always hold true. Without testing these explicitly, bugs can slip through that violate fundamental business constraints.

## Pattern

Test both that valid states can be created and that invalid states are prevented. Focus on constructor validation, state transitions, and business rule enforcement.

## Example

### ❌ Before - Missing Invariant Tests

```csharp
[Fact]
public void CreateOrder_Succeeds()
{
    var order = Order.Create(CustomerId.New(), Money.USD(100m));
    Assert.NotNull(order);
}
// ❌ What about negative amounts? Zero? Missing validation tests!
```

### ✅ After - Comprehensive Invariant Testing

```csharp
[Theory]
[InlineData(0.01)]
[InlineData(1)]
[InlineData(1000)]
[InlineData(999999.99)]
public void CreateOrder_WithValidAmount_Succeeds(decimal amount)
{
    // Arrange
    var customerId = CustomerId.New();
    var money = Money.USD(amount);

    // Act
    var result = Order.Create(customerId, money);

    // Assert
    Assert.True(result.IsSuccess);
}

[Theory]
[InlineData(0)]
[InlineData(-0.01)]
[InlineData(-100)]
public void CreateOrder_WithInvalidAmount_ReturnsFailure(decimal amount)
{
    // Arrange
    var customerId = CustomerId.New();
    var money = Money.USD(amount);

    // Act
    var result = Order.Create(customerId, money);

    // Assert
    Assert.True(result.IsFailure);
    Assert.Contains("positive", result.Error, StringComparison.OrdinalIgnoreCase);
}
```

## Testing Value Object Invariants

### Range Constraints

```csharp
public class AgeTests
{
    [Theory]
    [InlineData(0)]
    [InlineData(1)]
    [InlineData(18)]
    [InlineData(65)]
    [InlineData(120)]
    public void Create_WithValidAge_Succeeds(int value)
    {
        var result = Age.Create(value);
        
        Assert.True(result.IsSuccess);
        Assert.Equal(value, result.Value.Value);
    }

    [Theory]
    [InlineData(-1)]
    [InlineData(-100)]
    [InlineData(121)]
    [InlineData(200)]
    public void Create_WithInvalidAge_ReturnsFailure(int value)
    {
        var result = Age.Create(value);
        
        Assert.True(result.IsFailure);
    }
}
```

### Format Constraints

```csharp
public class EmailAddressTests
{
    [Theory]
    [InlineData("user@example.com")]
    [InlineData("user.name@example.com")]
    [InlineData("user+tag@example.com")]
    [InlineData("user@sub.example.com")]
    public void Create_WithValidFormat_Succeeds(string email)
    {
        var result = EmailAddress.Create(email);
        
        Assert.True(result.IsSuccess);
    }

    [Theory]
    [InlineData(null)]
    [InlineData("")]
    [InlineData("   ")]
    [InlineData("invalid")]
    [InlineData("@example.com")]
    [InlineData("user@")]
    [InlineData("user@@example.com")]
    public void Create_WithInvalidFormat_ReturnsFailure(string invalidEmail)
    {
        var result = EmailAddress.Create(invalidEmail);
        
        Assert.True(result.IsFailure);
    }
}
```

### Length Constraints

```csharp
public class ProductNameTests
{
    [Theory]
    [InlineData("A")]                                    // Minimum length
    [InlineData("Widget")]
    [InlineData("Super Premium Deluxe Product Name")]  
    [InlineData("123456789012345678901234567890123456789012345678901234567890")]  // Max 100
    public void Create_WithValidLength_Succeeds(string name)
    {
        var result = ProductName.Create(name);
        
        Assert.True(result.IsSuccess);
        Assert.Equal(name, result.Value.Value);
    }

    [Theory]
    [InlineData("")]
    [InlineData("   ")]
    [InlineData("This name is way too long and exceeds the maximum allowed length of 100 characters which should cause validation to fail and return an error")]
    public void Create_WithInvalidLength_ReturnsFailure(string name)
    {
        var result = ProductName.Create(name);
        
        Assert.True(result.IsFailure);
    }
}
```

## Testing Entity Invariants

### Aggregate Root Invariants

```csharp
public class OrderTests
{
    [Fact]
    public void CreateOrder_WithItems_CalculatesCorrectTotal()
    {
        // Invariant: Order total must equal sum of line items
        
        // Arrange
        var item1 = OrderItem.Create(ProductId.New(), quantity: 2, price: Money.USD(10m));
        var item2 = OrderItem.Create(ProductId.New(), quantity: 3, price: Money.USD(20m));

        // Act
        var order = Order.Create(CustomerId.New(), new[] { item1, item2 });

        // Assert
        var expectedTotal = Money.USD(80m); // (2 * $10) + (3 * $20)
        Assert.Equal(expectedTotal, order.Total);
    }

    [Fact]
    public void AddItem_UpdatesTotal()
    {
        // Invariant: Total must stay synchronized with items
        
        // Arrange
        var order = OrderMother.DefaultOrder();
        var initialTotal = order.Total;
        var newItem = OrderItem.Create(ProductId.New(), quantity: 1, price: Money.USD(50m));

        // Act
        order.AddItem(newItem);

        // Assert
        Assert.Equal(initialTotal + Money.USD(50m), order.Total);
    }

    [Fact]
    public void RemoveLastItem_PreventsEmptyOrder()
    {
        // Invariant: Order must have at least one item
        
        // Arrange
        var item = OrderItem.Create(ProductId.New(), quantity: 1, price: Money.USD(10m));
        var order = Order.Create(CustomerId.New(), new[] { item });

        // Act
        var result = order.RemoveItem(item.ProductId);

        // Assert
        Assert.True(result.IsFailure);
        Assert.Contains("at least one item", result.Error);
        Assert.Single(order.LineItems);
    }
}
```

### Consistency Boundaries

```csharp
public class BankAccountTests
{
    [Fact]
    public void Withdraw_ExceedingBalance_PreventedByInvariant()
    {
        // Invariant: Balance cannot go negative
        
        // Arrange
        var account = BankAccount.Create(AccountId.New(), Money.USD(100m));

        // Act
        var result = account.Withdraw(Money.USD(150m));

        // Assert
        Assert.True(result.IsFailure);
        Assert.Equal(Money.USD(100m), account.Balance); // Unchanged
    }

    [Fact]
    public void Deposit_MaintainsPositiveBalance()
    {
        // Invariant: Balance must be >= 0
        
        // Arrange
        var account = BankAccount.Create(AccountId.New(), Money.USD(0m));

        // Act
        account.Deposit(Money.USD(50m));

        // Assert
        Assert.Equal(Money.USD(50m), account.Balance);
        Assert.True(account.Balance >= Money.USD(0m));
    }
}
```

## Testing State Transition Invariants

```csharp
public class OrderStatusTests
{
    [Fact]
    public void TransitionToCancelled_FromPending_Allowed()
    {
        // Invariant: Pending orders can be cancelled
        
        var order = OrderMother.PendingOrder();
        
        var result = order.Cancel();
        
        Assert.True(result.IsSuccess);
        Assert.Equal(OrderStatus.Cancelled, order.Status);
    }

    [Fact]
    public void TransitionToCancelled_FromDelivered_Prevented()
    {
        // Invariant: Delivered orders cannot be cancelled
        
        var order = OrderMother.DeliveredOrder();
        
        var result = order.Cancel();
        
        Assert.True(result.IsFailure);
        Assert.Equal(OrderStatus.Delivered, order.Status); // Unchanged
    }

    [Fact]
    public void Ship_WithoutProcessing_Prevented()
    {
        // Invariant: Order must be processed before shipping
        
        var order = OrderMother.PendingOrder();
        
        var result = order.Ship(TrackingNumber.Create("TRACK123"));
        
        Assert.True(result.IsFailure);
        Assert.Contains("must be processed", result.Error);
    }
}
```

## Testing Business Rule Invariants

```csharp
public class DiscountTests
{
    [Fact]
    public void ApplyDiscount_ExceedingMaximum_ClampedToMaximum()
    {
        // Invariant: Discount cannot exceed 50% of order total
        
        // Arrange
        var order = OrderBuilder.Default()
            .WithSubtotal(Money.USD(100m))
            .Build();

        // Act
        order.ApplyDiscount(Percentage.FromDecimal(0.75m)); // 75%

        // Assert
        Assert.Equal(Money.USD(50m), order.Discount); // Clamped to 50%
        Assert.Equal(Money.USD(50m), order.Total); // 100 - 50
    }

    [Fact]
    public void ApplyMultipleDiscounts_SumNotExceedingMaximum()
    {
        // Invariant: Total discounts cannot exceed 50%
        
        // Arrange
        var order = OrderBuilder.Default()
            .WithSubtotal(Money.USD(100m))
            .Build();

        // Act
        order.ApplyDiscount(Percentage.FromDecimal(0.30m)); // 30%
        var result = order.ApplyDiscount(Percentage.FromDecimal(0.30m)); // +30%

        // Assert
        Assert.True(result.IsFailure);
        Assert.Contains("maximum", result.Error);
    }
}
```

## Testing Temporal Invariants

```csharp
public class ReservationTests
{
    [Fact]
    public void CreateReservation_EndBeforeStart_Prevented()
    {
        // Invariant: End time must be after start time
        
        // Arrange
        var start = DateTime.UtcNow.AddHours(2);
        var end = DateTime.UtcNow.AddHours(1); // Before start!

        // Act
        var result = Reservation.Create(ResourceId.New(), start, end);

        // Assert
        Assert.True(result.IsFailure);
        Assert.Contains("after", result.Error);
    }

    [Fact]
    public void CreateReservation_InPast_Prevented()
    {
        // Invariant: Cannot create reservations in the past
        
        // Arrange
        var start = DateTime.UtcNow.AddHours(-1);
        var end = DateTime.UtcNow.AddHours(1);

        // Act
        var result = Reservation.Create(ResourceId.New(), start, end);

        // Assert
        Assert.True(result.IsFailure);
        Assert.Contains("past", result.Error);
    }
}
```

## Testing Collection Invariants

```csharp
public class ShoppingCartTests
{
    [Fact]
    public void AddItem_ExceedingMaxQuantity_Prevented()
    {
        // Invariant: Cart items cannot exceed max quantity of 99
        
        // Arrange
        var cart = ShoppingCart.Create(CustomerId.New());
        var product = ProductId.New();

        // Act
        var result = cart.AddItem(product, quantity: 100);

        // Assert
        Assert.True(result.IsFailure);
        Assert.Contains("maximum quantity", result.Error);
    }

    [Fact]
    public void AddItems_ExceedingMaxItems_Prevented()
    {
        // Invariant: Cart cannot contain more than 50 unique items
        
        // Arrange
        var cart = ShoppingCart.Create(CustomerId.New());
        for (int i = 0; i < 50; i++)
        {
            cart.AddItem(ProductId.New(), quantity: 1);
        }

        // Act
        var result = cart.AddItem(ProductId.New(), quantity: 1);

        // Assert
        Assert.True(result.IsFailure);
        Assert.Contains("maximum", result.Error);
    }
}
```

## Testing Immutability Invariants

```csharp
public class MoneyTests
{
    [Fact]
    public void Operations_DoNotMutateOriginal()
    {
        // Invariant: Money is immutable
        
        // Arrange
        var original = Money.USD(100m);
        var originalAmount = original.Amount;

        // Act
        var added = original.Add(Money.USD(50m));
        var multiplied = original.Multiply(2m);

        // Assert
        Assert.Equal(originalAmount, original.Amount); // Unchanged
        Assert.Equal(Money.USD(150m), added);
        Assert.Equal(Money.USD(200m), multiplied);
    }
}
```

## Benefits

1. **Correctness**: Ensures business rules are always enforced
2. **Documentation**: Tests describe business constraints
3. **Regression prevention**: Catches violations of rules
4. **Confidence**: Know that invalid states can't exist
5. **Design feedback**: Hard-to-test invariants indicate design issues

## Guidelines

1. **Test both sides**: Valid and invalid cases
2. **Use property-based testing** for invariants
3. **Test state transitions** explicitly
4. **Verify invariants after operations**
5. **Make invalid states unrepresentable** in types

## See Also

- [Testing Value Objects](./testing-value-objects.md) — testing value semantics
- [Testing State Machines](./testing-state-machines.md) — state transition testing
- [Property-Based Testing](./property-based-testing.md) — invariant testing with generated data
- [Making Invalid States Unrepresentable](../patterns/making-invalid-states-unrepresentable.md) — design pattern
