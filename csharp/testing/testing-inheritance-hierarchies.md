# Testing Inheritance and Polymorphism

> Test inheritance hierarchies and polymorphic behavior—verify correct dispatch, override behavior, and substitutability.

## Problem

Object-oriented designs use inheritance and polymorphism. Tests must verify that derived classes correctly override base behavior and that polymorphic collections work correctly.

## Pattern

Test base class contracts, derived class specific behavior, and Liskov Substitution Principle compliance using polymorphic test patterns.

## Example

### ❌ Before - Duplicated Tests

```csharp
[Fact]
public void CreditCardPayment_ProcessPayment_CompletesTransaction()
{
    var payment = new CreditCardPayment("1234-5678", Money.USD(100m));
    var result = payment.Process();
    Assert.True(result.IsSuccess);
}

[Fact]
public void BankTransferPayment_ProcessPayment_CompletesTransaction()
{
    var payment = new BankTransferPayment("ACC-123", Money.USD(100m));
    var result = payment.Process();
    Assert.True(result.IsSuccess);
}

// Duplicating same test for each payment type
```

### ✅ After - Polymorphic Tests

```csharp
public abstract class PaymentMethodTests
{
    protected abstract PaymentMethod CreatePaymentMethod(Money amount);

    [Fact]
    public void ProcessPayment_ValidAmount_CompletesSuccessfully()
    {
        // Arrange
        var payment = CreatePaymentMethod(Money.USD(100m));

        // Act
        var result = payment.Process();

        // Assert
        result.IsSuccess.Should().BeTrue();
    }
}

public class CreditCardPaymentTests : PaymentMethodTests
{
    protected override PaymentMethod CreatePaymentMethod(Money amount) =>
        new CreditCardPayment("1234-5678", amount);

    [Fact]
    public void ProcessPayment_ExpiredCard_ReturnsFailure()
    {
        // Credit card specific test
    }
}

public class BankTransferPaymentTests : PaymentMethodTests
{
    protected override PaymentMethod CreatePaymentMethod(Money amount) =>
        new BankTransferPayment("ACC-123", amount);

    [Fact]
    public void ProcessPayment_InsufficientFunds_ReturnsFailure()
    {
        // Bank transfer specific test
    }
}
```

## Testing Base Class Contracts

### Abstract Base Class Tests

```csharp
public abstract class PaymentMethod
{
    public Money Amount { get; }
    public abstract Result<PaymentReceipt> Process();
    public abstract bool CanProcess();
}

public abstract class PaymentMethodContractTests
{
    protected abstract PaymentMethod CreateValid();
    protected abstract PaymentMethod CreateInvalid();

    [Fact]
    public void Process_ValidPayment_ReturnsSuccessWithReceipt()
    {
        // Arrange
        var payment = CreateValid();

        // Act
        var result = payment.Process();

        // Assert
        result.Should().BeSuccess();
        result.Value.Should().NotBeNull();
        result.Value.Amount.Should().Be(payment.Amount);
    }

    [Fact]
    public void Process_InvalidPayment_ReturnsFailure()
    {
        // Arrange
        var payment = CreateInvalid();

        // Act
        var result = payment.Process();

        // Assert
        result.Should().BeFailure();
    }

    [Fact]
    public void CanProcess_ValidPayment_ReturnsTrue()
    {
        // Arrange
        var payment = CreateValid();

        // Act
        var canProcess = payment.CanProcess();

        // Assert
        canProcess.Should().BeTrue();
    }
}
```

### Concrete Implementation Tests

```csharp
public class CreditCardPaymentContractTests : PaymentMethodContractTests
{
    protected override PaymentMethod CreateValid() =>
        CreditCardPayment.Create("4111-1111-1111-1111", "12/25", "123", Money.USD(100m));

    protected override PaymentMethod CreateInvalid() =>
        CreditCardPayment.Create("0000-0000-0000-0000", "01/20", "000", Money.USD(100m));

    [Fact]
    public void Create_InvalidCardNumber_ReturnsFailure()
    {
        // Arrange
        var invalidNumber = "invalid";

        // Act
        var result = CreditCardPayment.Create(invalidNumber, "12/25", "123", Money.USD(100m));

        // Assert
        result.Should().BeFailure();
        result.Error.Should().Contain("card number");
    }

    [Fact]
    public void Create_ExpiredCard_ReturnsFailure()
    {
        // Arrange
        var expiredDate = "01/20";

        // Act
        var result = CreditCardPayment.Create("4111-1111-1111-1111", expiredDate, "123", Money.USD(100m));

        // Assert
        result.Should().BeFailure();
        result.Error.Should().Contain("expired");
    }
}
```

## Testing Liskov Substitution Principle

### Behavioral Substitutability

```csharp
public class Shape
{
    public virtual double CalculateArea() => 0;
}

public class Rectangle : Shape
{
    public double Width { get; init; }
    public double Height { get; init; }

    public override double CalculateArea() => Width * Height;
}

public class Square : Shape
{
    public double Side { get; init; }

    public override double CalculateArea() => Side * Side;
}

[Theory]
[MemberData(nameof(GetShapes))]
public void CalculateArea_AnyShape_ReturnsNonNegative(Shape shape)
{
    // Act
    var area = shape.CalculateArea();

    // Assert
    area.Should().BeGreaterOrEqualTo(0);
}

[Theory]
[MemberData(nameof(GetShapes))]
public void CalculateArea_CalledTwice_ReturnsSameValue(Shape shape)
{
    // Act
    var area1 = shape.CalculateArea();
    var area2 = shape.CalculateArea();

    // Assert
    area1.Should().Be(area2);
}

public static IEnumerable<object[]> GetShapes()
{
    yield return new object[] { new Rectangle { Width = 10, Height = 5 } };
    yield return new object[] { new Square { Side = 10 } };
    yield return new object[] { new Circle { Radius = 5 } };
}
```

## Testing Virtual Methods

### Override Behavior

```csharp
public class OrderProcessor
{
    public virtual Result ValidateOrder(Order order)
    {
        if (order.Items.Count == 0)
            return Result.Failure("Order has no items");
            
        return Result.Success();
    }

    public virtual Result ProcessOrder(Order order)
    {
        var validation = ValidateOrder(order);
        if (validation.IsFailure)
            return validation;
            
        // Process...
        return Result.Success();
    }
}

public class PremiumOrderProcessor : OrderProcessor
{
    public override Result ValidateOrder(Order order)
    {
        // Call base validation first
        var baseValidation = base.ValidateOrder(order);
        if (baseValidation.IsFailure)
            return baseValidation;
            
        // Additional premium validation
        if (order.Total.Amount < Money.USD(100m).Amount)
            return Result.Failure("Premium orders require minimum $100");
            
        return Result.Success();
    }
}

[Fact]
public void ValidateOrder_EmptyOrder_BaseValidationFails()
{
    // Arrange
    var processor = new PremiumOrderProcessor();
    var order = Order.Create(CustomerId.New(), Array.Empty<OrderLineItem>());

    // Act
    var result = processor.ValidateOrder(order);

    // Assert
    result.Should().BeFailure();
    result.Error.Should().Contain("no items");
}

[Fact]
public void ValidateOrder_BelowMinimum_PremiumValidationFails()
{
    // Arrange
    var processor = new PremiumOrderProcessor();
    var order = CreateOrderWithTotal(Money.USD(50m));

    // Act
    var result = processor.ValidateOrder(order);

    // Assert
    result.Should().BeFailure();
    result.Error.Should().Contain("minimum $100");
}
```

## Testing Interface Implementations

### Multiple Implementations

```csharp
public interface INotificationSender
{
    Task<Result> SendAsync(Notification notification);
}

public abstract class NotificationSenderTests
{
    protected abstract INotificationSender CreateSender();
    protected abstract Notification CreateValidNotification();

    [Fact]
    public async Task SendAsync_ValidNotification_Succeeds()
    {
        // Arrange
        var sender = CreateSender();
        var notification = CreateValidNotification();

        // Act
        var result = await sender.SendAsync(notification);

        // Assert
        result.Should().BeSuccess();
    }

    [Fact]
    public async Task SendAsync_NullNotification_ThrowsArgumentNullException()
    {
        // Arrange
        var sender = CreateSender();

        // Act
        Func<Task> act = async () => await sender.SendAsync(null!);

        // Assert
        await act.Should().ThrowAsync<ArgumentNullException>();
    }
}

public class EmailSenderTests : NotificationSenderTests
{
    protected override INotificationSender CreateSender() => new EmailSender();
    protected override Notification CreateValidNotification() => 
        new EmailNotification("test@example.com", "Subject", "Body");
}

public class SmsSenderTests : NotificationSenderTests
{
    protected override INotificationSender CreateSender() => new SmsSender();
    protected override Notification CreateValidNotification() =>
        new SmsNotification("+1234567890", "Message");
}
```

## Testing Polymorphic Collections

### Heterogeneous Collections

```csharp
[Fact]
public void ProcessPayments_MixedTypes_AllProcessCorrectly()
{
    // Arrange
    var payments = new PaymentMethod[]
    {
        new CreditCardPayment("4111-1111", Money.USD(100m)),
        new BankTransferPayment("ACC-123", Money.USD(200m)),
        new PayPalPayment("user@example.com", Money.USD(150m))
    };

    var processor = new PaymentProcessor();

    // Act
    var results = payments.Select(p => processor.Process(p)).ToList();

    // Assert
    results.Should().AllSatisfy(r => r.IsSuccess.Should().BeTrue());
}

[Fact]
public void CalculateTotalFees_MixedPaymentTypes_SumsCorrectly()
{
    // Arrange
    var payments = new PaymentMethod[]
    {
        new CreditCardPayment("4111-1111", Money.USD(100m)),  // 3% fee
        new BankTransferPayment("ACC-123", Money.USD(200m)),  // $5 fee
        new CashPayment(Money.USD(50m))                       // no fee
    };

    // Act
    var totalFees = payments.Sum(p => p.CalculateFee().Amount);

    // Assert
    totalFees.Should().Be(8m);  // (100 * 0.03) + 5 + 0
}
```

## Testing Abstract Properties

### Property Contracts

```csharp
public abstract class Entity
{
    public abstract Guid Id { get; }
    public abstract DateTime CreatedAt { get; }
    public abstract DateTime? UpdatedAt { get; }
}

public abstract class EntityContractTests<T> where T : Entity
{
    protected abstract T CreateEntity();

    [Fact]
    public void Id_AfterCreation_IsNotEmpty()
    {
        // Arrange
        var entity = CreateEntity();

        // Assert
        entity.Id.Should().NotBeEmpty();
    }

    [Fact]
    public void CreatedAt_AfterCreation_IsRecentUtcTime()
    {
        // Arrange
        var before = DateTime.UtcNow;
        
        // Act
        var entity = CreateEntity();
        
        var after = DateTime.UtcNow;

        // Assert
        entity.CreatedAt.Should().BeOnOrAfter(before);
        entity.CreatedAt.Should().BeOnOrBefore(after);
    }

    [Fact]
    public void UpdatedAt_OnNewEntity_IsNull()
    {
        // Arrange
        var entity = CreateEntity();

        // Assert
        entity.UpdatedAt.Should().BeNull();
    }
}

public class OrderEntityTests : EntityContractTests<Order>
{
    protected override Order CreateEntity() =>
        Order.Create(CustomerId.New(), CreateOrderItems());
}
```

## Testing Sealed vs Virtual Methods

### Preventing Override

```csharp
public class PaymentBase
{
    // Sealed - cannot be overridden
    public sealed Money CalculateFinalAmount()
    {
        var amount = GetBaseAmount();
        var fee = CalculateFee();
        return amount.Add(fee);
    }

    // Virtual - can be overridden
    public virtual Money GetBaseAmount() => Amount;
    
    // Virtual - can be overridden
    public virtual Money CalculateFee() => Money.USD(0);

    public Money Amount { get; init; }
}

[Fact]
public void CalculateFinalAmount_InDerivedClass_UsesBaseLogic()
{
    // Arrange
    var payment = new CreditCardPayment
    {
        Amount = Money.USD(100m)
    };
    
    // Override CalculateFee to return 3% of amount
    // CalculateFinalAmount is sealed, so it still adds fee to amount

    // Act
    var final = payment.CalculateFinalAmount();

    // Assert
    final.Should().Be(Money.USD(103m));  // 100 + 3% fee
}
```

## Type Checking and Casting

### Runtime Type Tests

```csharp
[Fact]
public void ProcessPayment_DifferentTypes_HandlesCorrectly()
{
    // Arrange
    var payments = new PaymentMethod[]
    {
        new CreditCardPayment("4111-1111", Money.USD(100m)),
        new BankTransferPayment("ACC-123", Money.USD(200m))
    };

    // Act & Assert
    foreach (var payment in payments)
    {
        payment.Should().BeAssignableTo<PaymentMethod>();
        
        if (payment is CreditCardPayment cc)
        {
            cc.CardNumber.Should().NotBeNullOrEmpty();
        }
        else if (payment is BankTransferPayment bt)
        {
            bt.AccountNumber.Should().NotBeNullOrEmpty();
        }
    }
}

[Fact]
public void CastPayment_ToSpecificType_PreservesData()
{
    // Arrange
    PaymentMethod payment = new CreditCardPayment("4111-1111", Money.USD(100m));

    // Act
    var creditCard = payment as CreditCardPayment;

    // Assert
    creditCard.Should().NotBeNull();
    creditCard!.CardNumber.Should().Be("4111-1111");
}
```

## Guidelines

1. **Test base contracts**: Create abstract test classes for base behavior
2. **Test substitutability**: Verify derived classes work wherever base is used
3. **Test specific behavior**: Add tests for derived class unique features
4. **Use factory methods**: Abstract test classes use factories for polymorphic creation
5. **Test collections**: Verify heterogeneous collections work correctly

## Benefits

1. **DRY tests**: Base tests run for all implementations
2. **Contract enforcement**: Ensure all implementations meet interface contracts
3. **LSP verification**: Confirm substitutability works correctly
4. **Comprehensive coverage**: Test both shared and specific behavior
5. **Maintenance**: Add new implementations easily with guaranteed coverage

## See Also

- [Testing Discriminated Unions](./testing-discriminated-unions.md) — testing sum types
- [Testing State Machines](./testing-state-machines.md) — testing state hierarchies
- [Testing Factory Methods](./testing-factory-methods.md) — polymorphic construction
- [Test Data Builders](./test-data-builders.md) — building polymorphic test data
