# Testing Factory Methods

> Verify factory method behavior, object creation, validation, and error handling using MSTest and FluentAssertions.

## Problem

Factory methods encapsulate complex object creation logic, including validation and initialization. Tests must verify correct object creation, validation rules, and failure scenarios.

## Pattern

Test factory methods for successful creation, validation failures, default values, and factory method overloads.

## Example

### ❌ Before - Incomplete Factory Testing

```csharp
[TestMethod]
public void Create_ReturnsCustomer()
{
    var customer = Customer.Create("John", "Doe");
    Assert.IsNotNull(customer);
    // ❌ Missing: validation, failure cases, property verification
}
```

### ✅ After - Comprehensive Factory Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

public record CustomerId
{
    public Guid Value { get; }

    private CustomerId(Guid value) => Value = value;

    public static CustomerId New() => new(Guid.NewGuid());
    public static CustomerId From(Guid value) => new(value);
}

public class Customer
{
    public CustomerId Id { get; }
    public string FirstName { get; }
    public string LastName { get; }
    public string Email { get; }

    private Customer(CustomerId id, string firstName, string lastName, string email)
    {
        Id = id;
        FirstName = firstName;
        LastName = lastName;
        Email = email;
    }

    public static Result<Customer> Create(string firstName, string lastName, string email)
    {
        if (string.IsNullOrWhiteSpace(firstName))
            return Result.Failure<Customer>("First name is required");

        if (string.IsNullOrWhiteSpace(lastName))
            return Result.Failure<Customer>("Last name is required");

        if (string.IsNullOrWhiteSpace(email) || !email.Contains("@"))
            return Result.Failure<Customer>("Valid email is required");

        return Result.Success(new Customer(CustomerId.New(), firstName, lastName, email));
    }
}

[TestClass]
public class CustomerFactoryTests
{
    [TestMethod]
    public void Create_WithValidData_ReturnsSuccess()
    {
        // Arrange
        var firstName = "John";
        var lastName = "Doe";
        var email = "john@example.com";

        // Act
        var result = Customer.Create(firstName, lastName, email);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeNull();
        result.Value.FirstName.Should().Be(firstName);
        result.Value.LastName.Should().Be(lastName);
        result.Value.Email.Should().Be(email);
        result.Value.Id.Should().NotBeNull();
    }

    [TestMethod]
    public void Create_WithEmptyFirstName_ReturnsFailure()
    {
        // Arrange
        var firstName = "";
        var lastName = "Doe";
        var email = "john@example.com";

        // Act
        var result = Customer.Create(firstName, lastName, email);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("First name");
    }

    [TestMethod]
    public void Create_WithInvalidEmail_ReturnsFailure()
    {
        // Arrange
        var firstName = "John";
        var lastName = "Doe";
        var email = "invalid-email";

        // Act
        var result = Customer.Create(firstName, lastName, email);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("email");
    }

    [TestMethod]
    public void Create_GeneratesUniqueIds()
    {
        // Arrange & Act
        var customer1 = Customer.Create("John", "Doe", "john@example.com").Value;
        var customer2 = Customer.Create("Jane", "Smith", "jane@example.com").Value;

        // Assert
        customer1.Id.Should().NotBe(customer2.Id);
    }
}
```

## Testing Static Factory Methods

```csharp
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
            return Result.Failure<Money>("Currency must be 3 characters");

        return Result.Success(new Money(amount, currency.ToUpperInvariant()));
    }

    public static Money USD(decimal amount) => Create(amount, "USD").Value;
    public static Money EUR(decimal amount) => Create(amount, "EUR").Value;
    public static Money Zero(string currency) => new(0, currency);
}

[TestClass]
public class MoneyFactoryTests
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
    public void Create_NormalizeCurrencyToUpperCase()
    {
        // Act
        var result = Money.Create(100m, "usd");

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Currency.Should().Be("USD");
    }

    [TestMethod]
    public void USD_CreatesUSDMoney()
    {
        // Act
        var money = Money.USD(50m);

        // Assert
        money.Amount.Should().Be(50m);
        money.Currency.Should().Be("USD");
    }

    [TestMethod]
    public void EUR_CreatesEURMoney()
    {
        // Act
        var money = Money.EUR(75m);

        // Assert
        money.Amount.Should().Be(75m);
        money.Currency.Should().Be("EUR");
    }

    [TestMethod]
    public void Zero_CreatesZeroAmount()
    {
        // Act
        var money = Money.Zero("GBP");

        // Assert
        money.Amount.Should().Be(0m);
        money.Currency.Should().Be("GBP");
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
    public void Create_WithInvalidCurrencyLength_ReturnsFailure()
    {
        // Act
        var result = Money.Create(100m, "US");

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("3 characters");
    }
}
```

## Testing Builder Pattern with Factories

```csharp
public class OrderBuilder
{
    private CustomerId? _customerId;
    private Money? _total;
    private readonly List<OrderLine> _lines = new();

    public OrderBuilder WithCustomer(CustomerId customerId)
    {
        _customerId = customerId;
        return this;
    }

    public OrderBuilder WithTotal(Money total)
    {
        _total = total;
        return this;
    }

    public OrderBuilder AddLine(OrderLine line)
    {
        _lines.Add(line);
        return this;
    }

    public Result<Order> Build()
    {
        if (_customerId is null)
            return Result.Failure<Order>("Customer ID is required");

        if (_total is null)
            return Result.Failure<Order>("Total is required");

        if (_lines.Count == 0)
            return Result.Failure<Order>("At least one order line is required");

        return Order.Create(OrderId.New(), _customerId, _total, _lines);
    }
}

[TestClass]
public class OrderBuilderFactoryTests
{
    [TestMethod]
    public void Build_WithAllRequiredData_ReturnsSuccess()
    {
        // Arrange
        var builder = new OrderBuilder()
            .WithCustomer(CustomerId.New())
            .WithTotal(Money.USD(100m))
            .AddLine(CreateOrderLine());

        // Act
        var result = builder.Build();

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeNull();
    }

    [TestMethod]
    public void Build_WithoutCustomerId_ReturnsFailure()
    {
        // Arrange
        var builder = new OrderBuilder()
            .WithTotal(Money.USD(100m))
            .AddLine(CreateOrderLine());

        // Act
        var result = builder.Build();

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("Customer ID");
    }

    [TestMethod]
    public void Build_WithoutOrderLines_ReturnsFailure()
    {
        // Arrange
        var builder = new OrderBuilder()
            .WithCustomer(CustomerId.New())
            .WithTotal(Money.USD(100m));

        // Act
        var result = builder.Build();

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("order line");
    }

    [TestMethod]
    public void Build_FluentInterface_ReturnsBuilder()
    {
        // Arrange
        var builder = new OrderBuilder();

        // Act
        var result = builder
            .WithCustomer(CustomerId.New())
            .WithTotal(Money.USD(100m));

        // Assert
        result.Should().BeSameAs(builder);
    }
}
```

## Testing Abstract Factory Pattern

```csharp
public interface IPaymentMethodFactory
{
    Result<IPaymentMethod> CreatePaymentMethod(string type, Dictionary<string, string> parameters);
}

public class PaymentMethodFactory : IPaymentMethodFactory
{
    public Result<IPaymentMethod> CreatePaymentMethod(string type, Dictionary<string, string> parameters)
    {
        return type.ToLowerInvariant() switch
        {
            "creditcard" => CreateCreditCard(parameters),
            "banktransfer" => CreateBankTransfer(parameters),
            "paypal" => CreatePayPal(parameters),
            _ => Result.Failure<IPaymentMethod>($"Unknown payment type: {type}")
        };
    }

    private Result<IPaymentMethod> CreateCreditCard(Dictionary<string, string> parameters)
    {
        if (!parameters.TryGetValue("cardNumber", out var cardNumber))
            return Result.Failure<IPaymentMethod>("Card number is required");

        return Result.Success<IPaymentMethod>(new CreditCard(cardNumber));
    }

    // Other factory methods...
}

[TestClass]
public class AbstractFactoryTests
{
    [TestMethod]
    public void CreatePaymentMethod_CreditCard_ReturnsCorrectType()
    {
        // Arrange
        var factory = new PaymentMethodFactory();
        var parameters = new Dictionary<string, string>
        {
            ["cardNumber"] = "1234-5678-9012-3456"
        };

        // Act
        var result = factory.CreatePaymentMethod("creditcard", parameters);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().BeOfType<CreditCard>();
    }

    [TestMethod]
    public void CreatePaymentMethod_UnknownType_ReturnsFailure()
    {
        // Arrange
        var factory = new PaymentMethodFactory();
        var parameters = new Dictionary<string, string>();

        // Act
        var result = factory.CreatePaymentMethod("unknown", parameters);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("Unknown payment type");
    }

    [TestMethod]
    public void CreatePaymentMethod_MissingParameters_ReturnsFailure()
    {
        // Arrange
        var factory = new PaymentMethodFactory();
        var parameters = new Dictionary<string, string>();

        // Act
        var result = factory.CreatePaymentMethod("creditcard", parameters);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("required");
    }
}
```

## Testing Factory Registration and Dependencies

```csharp
public class ServiceFactory
{
    private readonly Dictionary<Type, Func<object>> _factories = new();

    public void Register<T>(Func<T> factory)
    {
        _factories[typeof(T)] = () => factory()!;
    }

    public T Create<T>()
    {
        if (!_factories.TryGetValue(typeof(T), out var factory))
            throw new InvalidOperationException($"No factory registered for {typeof(T).Name}");

        return (T)factory();
    }
}

[TestClass]
public class FactoryRegistrationTests
{
    [TestMethod]
    public void Create_WithRegisteredFactory_ReturnsInstance()
    {
        // Arrange
        var factory = new ServiceFactory();
        factory.Register<ICustomerRepository>(() => new InMemoryCustomerRepository());

        // Act
        var repository = factory.Create<ICustomerRepository>();

        // Assert
        repository.Should().NotBeNull();
        repository.Should().BeOfType<InMemoryCustomerRepository>();
    }

    [TestMethod]
    public void Create_WithoutRegistration_ThrowsException()
    {
        // Arrange
        var factory = new ServiceFactory();

        // Act
        Action act = () => factory.Create<ICustomerRepository>();

        // Assert
        act.Should().Throw<InvalidOperationException>()
            .WithMessage("*No factory registered*");
    }

    [TestMethod]
    public void Create_CallsFactoryEachTime()
    {
        // Arrange
        var factory = new ServiceFactory();
        factory.Register<ICustomerRepository>(() => new InMemoryCustomerRepository());

        // Act
        var instance1 = factory.Create<ICustomerRepository>();
        var instance2 = factory.Create<ICustomerRepository>();

        // Assert
        instance1.Should().NotBeSameAs(instance2);
    }
}
```

## Why It's Important

1. **Encapsulation**: Factory methods hide creation complexity
2. **Validation**: Centralized validation logic
3. **Consistency**: Ensure objects are created correctly
4. **Type Safety**: Return types enforce contracts
5. **Testability**: Easy to verify creation logic

## Guidelines

**Testing Factory Methods**
- Test successful creation with valid inputs
- Test all validation rules and failure cases
- Verify generated values (IDs, timestamps)
- Test factory method overloads
- Verify object state after creation

**Common Patterns to Test**
- Static factory methods
- Builder pattern build methods
- Abstract factory implementations
- Factory registration and resolution
- Named constructors

**Assertions**
- Verify return type is correct
- Check all properties are set
- Validate error messages for failures
- Ensure uniqueness where required

## Common Pitfalls

❌ **Only testing happy path**
```csharp
// Incomplete
var result = Customer.Create("John", "Doe", "john@example.com");
result.IsSuccess.Should().BeTrue();
```

✅ **Test validation failures**
```csharp
// Complete
Customer.Create("", "Doe", "john@example.com")
    .IsFailure.Should().BeTrue();
Customer.Create("John", "", "john@example.com")
    .IsFailure.Should().BeTrue();
Customer.Create("John", "Doe", "invalid")
    .IsFailure.Should().BeTrue();
```

❌ **Not verifying all properties**
```csharp
// Incomplete
result.Value.Should().NotBeNull();
```

✅ **Verify complete state**
```csharp
// Complete
result.Value.FirstName.Should().Be("John");
result.Value.LastName.Should().Be("Doe");
result.Value.Email.Should().Be("john@example.com");
result.Value.Id.Should().NotBeNull();
```

## See Also

- [Testing Smart Constructors](./testing-smart-constructors.md) — validated construction
- [Static Factory Methods](../patterns/static-factory-methods.md) — factory pattern
- [Domain Factories](../patterns/domain-factories.md) — domain object creation
- [Builder Pattern](../patterns/type-safe-builder.md) — complex object construction
