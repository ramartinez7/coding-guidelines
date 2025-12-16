# Testing Reference Types and Null

> Verify reference type behavior, null handling, and object identity using FluentAssertions.

## Problem

Reference types require testing for null safety, object identity vs equality, type checking, and proper initialization. Nullable reference types add additional verification needs.

## Pattern

Use FluentAssertions to test null handling, reference equality, type assertions, and nullable reference type behavior.

## Example

### ❌ Before - Basic Null Testing

```csharp
[TestMethod]
public void CreateCustomer_ReturnsNonNull()
{
    var customer = factory.Create();
    Assert.IsNotNull(customer);
    // ❌ Missing: type check, property verification
}
```

### ✅ After - Comprehensive Reference Type Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class CustomerFactoryTests
{
    [TestMethod]
    public void Create_ReturnsNonNullCustomer()
    {
        // Arrange
        var factory = new CustomerFactory();

        // Act
        var customer = factory.Create("John", "Doe");

        // Assert
        customer.Should().NotBeNull();
        customer.Should().BeOfType<Customer>();
    }

    [TestMethod]
    public void Create_WithNullName_ReturnsFailure()
    {
        // Arrange
        var factory = new CustomerFactory();

        // Act
        var result = factory.Create(null, "Doe");

        // Assert
        result.Should().BeNull()
            .Or.Subject.As<Result<Customer>>().IsFailure.Should().BeTrue();
    }

    [TestMethod]
    public void Create_ReturnsDifferentInstances()
    {
        // Arrange
        var factory = new CustomerFactory();

        // Act
        var customer1 = factory.Create("John", "Doe");
        var customer2 = factory.Create("John", "Doe");

        // Assert
        customer1.Should().NotBeSameAs(customer2);
    }
}
```

## Testing Null and Nullability

```csharp
public class CustomerService
{
    public Customer? FindById(CustomerId id) => 
        _repository.Find(id);

    public IEnumerable<Customer> GetAll() => 
        _repository.GetAll() ?? Enumerable.Empty<Customer>();
}

[TestClass]
public class CustomerServiceNullTests
{
    [TestMethod]
    public void FindById_WhenNotFound_ReturnsNull()
    {
        // Arrange
        var service = CreateServiceWithEmptyRepository();
        var customerId = CustomerId.New();

        // Act
        var result = service.FindById(customerId);

        // Assert
        result.Should().BeNull();
    }

    [TestMethod]
    public void FindById_WhenFound_ReturnsNonNull()
    {
        // Arrange
        var customerId = CustomerId.New();
        var service = CreateServiceWithCustomer(customerId);

        // Act
        var result = service.FindById(customerId);

        // Assert
        result.Should().NotBeNull();
        result.Id.Should().Be(customerId);
    }

    [TestMethod]
    public void GetAll_WhenEmpty_ReturnsEmptyNotNull()
    {
        // Arrange
        var service = CreateServiceWithEmptyRepository();

        // Act
        var result = service.GetAll();

        // Assert
        result.Should().NotBeNull();
        result.Should().BeEmpty();
    }
}
```

## Testing Object Identity vs Equality

```csharp
[TestClass]
public class ObjectIdentityTests
{
    [TestMethod]
    public void TwoReferences_ToSameObject_AreSame()
    {
        // Arrange
        var customer = Customer.Create("John", "Doe");
        var reference1 = customer;
        var reference2 = customer;

        // Act & Assert
        reference1.Should().BeSameAs(reference2);
        ReferenceEquals(reference1, reference2).Should().BeTrue();
    }

    [TestMethod]
    public void TwoInstances_WithSameValues_AreNotSame()
    {
        // Arrange
        var customer1 = Customer.Create("John", "Doe");
        var customer2 = Customer.Create("John", "Doe");

        // Act & Assert
        customer1.Should().NotBeSameAs(customer2);
        customer1.Should().Be(customer2); // If value equality is implemented
    }

    [TestMethod]
    public void Clone_CreatesDifferentInstance()
    {
        // Arrange
        var original = Customer.Create("John", "Doe");

        // Act
        var clone = original.Clone();

        // Assert
        clone.Should().NotBeSameAs(original);
        clone.Should().BeEquivalentTo(original);
    }
}
```

## Testing Type Assertions

```csharp
public abstract class PaymentMethod { }
public class CreditCard : PaymentMethod { }
public class BankTransfer : PaymentMethod { }

[TestClass]
public class TypeAssertionTests
{
    [TestMethod]
    public void GetPaymentMethod_ReturnsCreditCardType()
    {
        // Arrange
        var service = new PaymentService();

        // Act
        var payment = service.GetPaymentMethod("credit-card");

        // Assert
        payment.Should().NotBeNull();
        payment.Should().BeOfType<CreditCard>();
        payment.Should().BeAssignableTo<PaymentMethod>();
    }

    [TestMethod]
    public void GetPaymentMethod_CanBeDowncast()
    {
        // Arrange
        var service = new PaymentService();

        // Act
        var payment = service.GetPaymentMethod("credit-card");

        // Assert
        payment.Should().BeAssignableTo<CreditCard>();
        var creditCard = payment as CreditCard;
        creditCard.Should().NotBeNull();
    }

    [TestMethod]
    public void PaymentMethod_IsNotWrongType()
    {
        // Arrange
        var payment = new CreditCard();

        // Act & Assert
        payment.Should().NotBeOfType<BankTransfer>();
        payment.Should().BeOfType<CreditCard>();
    }
}
```

## Testing Nullable Reference Types

```csharp
#nullable enable

public class OrderValidator
{
    public string? Validate(Order? order)
    {
        if (order is null)
            return "Order cannot be null";

        if (order.Total.Amount <= 0)
            return "Order total must be positive";

        return null; // Valid
    }
}

[TestClass]
public class NullableReferenceTests
{
    [TestMethod]
    public void Validate_WithNullOrder_ReturnsErrorMessage()
    {
        // Arrange
        var validator = new OrderValidator();

        // Act
        var result = validator.Validate(null);

        // Assert
        result.Should().NotBeNull();
        result.Should().Contain("null");
    }

    [TestMethod]
    public void Validate_WithValidOrder_ReturnsNull()
    {
        // Arrange
        var validator = new OrderValidator();
        var order = CreateValidOrder();

        // Act
        var result = validator.Validate(order);

        // Assert
        result.Should().BeNull();
    }

    [TestMethod]
    public void Validate_WithInvalidOrder_ReturnsErrorMessage()
    {
        // Arrange
        var validator = new OrderValidator();
        var order = CreateOrderWithZeroTotal();

        // Act
        var result = validator.Validate(order);

        // Assert
        result.Should().NotBeNull();
        result.Should().Contain("positive");
    }
}
```

## Testing Initialization and Default Values

```csharp
[TestClass]
public class InitializationTests
{
    [TestMethod]
    public void NewObject_HasDefaultProperties()
    {
        // Arrange & Act
        var config = new Configuration();

        // Assert
        config.Should().NotBeNull();
        config.Timeout.Should().Be(TimeSpan.FromSeconds(30));
        config.ConnectionString.Should().NotBeNullOrEmpty();
    }

    [TestMethod]
    public void ObjectInitializer_SetsProperties()
    {
        // Arrange & Act
        var customer = new Customer
        {
            FirstName = "John",
            LastName = "Doe",
            Email = "john@example.com"
        };

        // Assert
        customer.FirstName.Should().Be("John");
        customer.LastName.Should().Be("Doe");
        customer.Email.Should().NotBeNullOrEmpty();
    }

    [TestMethod]
    public void RequiredProperties_AreNotNull()
    {
        // Arrange
        var customer = CreateCustomer();

        // Assert
        customer.Id.Should().NotBeNull();
        customer.FirstName.Should().NotBeNull();
        customer.LastName.Should().NotBeNull();
        customer.Email.Should().NotBeNull();
    }
}
```

## Testing Null Object Pattern

```csharp
public interface ILogger
{
    void Log(string message);
}

public class NullLogger : ILogger
{
    public void Log(string message) { } // No-op
}

[TestClass]
public class NullObjectPatternTests
{
    [TestMethod]
    public void NullLogger_CanBeUsedSafely()
    {
        // Arrange
        var logger = new NullLogger();

        // Act
        Action act = () => logger.Log("Test message");

        // Assert
        act.Should().NotThrow();
    }

    [TestMethod]
    public void NullLogger_IsNotNull()
    {
        // Arrange
        ILogger logger = new NullLogger();

        // Assert
        logger.Should().NotBeNull();
        logger.Should().BeOfType<NullLogger>();
    }
}
```

## Testing Null Propagation

```csharp
[TestClass]
public class NullPropagationTests
{
    [TestMethod]
    public void NullConditionalOperator_WithNull_ReturnsNull()
    {
        // Arrange
        Customer? customer = null;

        // Act
        var email = customer?.Email;

        // Assert
        email.Should().BeNull();
    }

    [TestMethod]
    public void NullConditionalOperator_WithObject_ReturnsValue()
    {
        // Arrange
        var customer = CreateCustomer();

        // Act
        var email = customer?.Email;

        // Assert
        email.Should().NotBeNull();
        email.Should().Be(customer.Email);
    }

    [TestMethod]
    public void NullCoalescingOperator_WithNull_ReturnsDefault()
    {
        // Arrange
        Customer? customer = null;

        // Act
        var email = customer?.Email ?? "default@example.com";

        // Assert
        email.Should().Be("default@example.com");
    }
}
```

## Why It's Important

1. **Null Safety**: Prevent null reference exceptions
2. **Type Safety**: Ensure correct types at runtime
3. **Identity**: Distinguish object identity from equality
4. **Initialization**: Verify proper object setup
5. **Nullable Types**: Test nullable reference type contracts

## Guidelines

**Null Testing**
- `.Should().BeNull()` - assert null
- `.Should().NotBeNull()` - assert not null
- `.Should().NotBeNullOrEmpty()` - for strings/collections

**Type Testing**
- `.Should().BeOfType<T>()` - exact type match
- `.Should().BeAssignableTo<T>()` - type compatibility
- `.Should().NotBeOfType<T>()` - type exclusion

**Reference Testing**
- `.Should().BeSameAs()` - reference equality
- `.Should().NotBeSameAs()` - different instances
- `.Should().BeEquivalentTo()` - value equality

## Common Pitfalls

❌ **Not checking for null**
```csharp
// Missing null check
var customer = service.Find(id);
customer.Email.Should().NotBeEmpty(); // NullReferenceException!
```

✅ **Always check null first**
```csharp
// Safe
var customer = service.Find(id);
customer.Should().NotBeNull();
customer!.Email.Should().NotBeEmpty();
```

❌ **Confusing identity and equality**
```csharp
// Wrong - tests reference equality
customer1.Should().BeSameAs(customer2);
```

✅ **Use appropriate equality**
```csharp
// Correct - tests value equality
customer1.Should().BeEquivalentTo(customer2);
```

## See Also

- [Testing Nullable Types](./testing-nullable-types.md) — nullable value types
- [Nullable Reference Types](../patterns/nullable-reference-types.md) — null safety
- [Null Object Pattern](../patterns/null-object-pattern.md) — avoiding nulls
- [Option Monad](../patterns/option-monad.md) — explicit optionality
