# Testing Type-Safe Builders

> Verify builder pattern implementation, fluent APIs, immutability, and build validation using MSTest and FluentAssertions.

## Problem

Builders construct complex objects step-by-step through fluent interfaces. Tests must verify builder state, fluent chaining, build validation, and that builders produce correctly configured objects.

## Pattern

Test builder fluent interface, required vs optional properties, build validation, immutability, and builder reusability.

## Example

### ❌ Before - No Builder Testing

```csharp
[TestMethod]
public void BuildOrder_Works()
{
    var builder = new OrderBuilder();
    var order = builder.Build();
    Assert.IsNotNull(order);
    // ❌ Doesn't verify builder behavior
}
```

### ✅ After - Comprehensive Builder Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

public class OrderBuilder
{
    private CustomerId? _customerId;
    private Money? _total;
    private OrderStatus _status = OrderStatus.Pending;
    private readonly List<OrderLine> _lines = new();
    private string? _notes;

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

    public OrderBuilder WithStatus(OrderStatus status)
    {
        _status = status;
        return this;
    }

    public OrderBuilder AddLine(OrderLine line)
    {
        _lines.Add(line);
        return this;
    }

    public OrderBuilder WithNotes(string notes)
    {
        _notes = notes;
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

        return Order.Create(
            OrderId.New(),
            _customerId,
            _total,
            _lines,
            _status,
            _notes
        );
    }
}

[TestClass]
public class OrderBuilderTests
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
    public void Build_WithoutTotal_ReturnsFailure()
    {
        // Arrange
        var builder = new OrderBuilder()
            .WithCustomer(CustomerId.New())
            .AddLine(CreateOrderLine());

        // Act
        var result = builder.Build();

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Contain("Total");
    }

    [TestMethod]
    public void Build_WithoutLines_ReturnsFailure()
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
    public void FluentInterface_ReturnsSelf()
    {
        // Arrange
        var builder = new OrderBuilder();

        // Act & Assert
        builder.WithCustomer(CustomerId.New()).Should().BeSameAs(builder);
        builder.WithTotal(Money.USD(100m)).Should().BeSameAs(builder);
        builder.WithStatus(OrderStatus.Pending).Should().BeSameAs(builder);
    }

    [TestMethod]
    public void Build_SetsOptionalProperties()
    {
        // Arrange
        var notes = "Special handling required";
        var builder = new OrderBuilder()
            .WithCustomer(CustomerId.New())
            .WithTotal(Money.USD(100m))
            .AddLine(CreateOrderLine())
            .WithNotes(notes);

        // Act
        var result = builder.Build();

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Notes.Should().Be(notes);
    }

    [TestMethod]
    public void Build_WithDefaultStatus_UsesPending()
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
        result.Value.Status.Should().Be(OrderStatus.Pending);
    }
}
```

## Testing Builder Reusability

```csharp
[TestClass]
public class BuilderReusabilityTests
{
    [TestMethod]
    public void Builder_CanBuildMultipleTimes()
    {
        // Arrange
        var builder = new OrderBuilder()
            .WithCustomer(CustomerId.New())
            .WithTotal(Money.USD(100m))
            .AddLine(CreateOrderLine());

        // Act
        var result1 = builder.Build();
        var result2 = builder.Build();

        // Assert
        result1.IsSuccess.Should().BeTrue();
        result2.IsSuccess.Should().BeTrue();
        result1.Value.Id.Should().NotBe(result2.Value.Id); // Different instances
    }

    [TestMethod]
    public void Builder_CanBeModifiedAfterBuild()
    {
        // Arrange
        var customerId1 = CustomerId.New();
        var customerId2 = CustomerId.New();

        var builder = new OrderBuilder()
            .WithCustomer(customerId1)
            .WithTotal(Money.USD(100m))
            .AddLine(CreateOrderLine());

        // Act
        var result1 = builder.Build();
        builder.WithCustomer(customerId2);
        var result2 = builder.Build();

        // Assert
        result1.Value.CustomerId.Should().Be(customerId1);
        result2.Value.CustomerId.Should().Be(customerId2);
    }
}
```

## Testing Type-Safe Builder Pattern

```csharp
// Type-safe builder using phantom types
public interface ICustomerSet { }
public interface IEmailSet { }

public class TypeSafeCustomerBuilder<TCustomerSet, TEmailSet>
{
    private string? _firstName;
    private string? _lastName;
    private string? _email;

    private TypeSafeCustomerBuilder() { }

    public static TypeSafeCustomerBuilder<object, object> Create() =>
        new TypeSafeCustomerBuilder<object, object>();

    public TypeSafeCustomerBuilder<ICustomerSet, TEmailSet> WithName(string firstName, string lastName)
    {
        var builder = new TypeSafeCustomerBuilder<ICustomerSet, TEmailSet>
        {
            _firstName = firstName,
            _lastName = lastName,
            _email = this._email
        };
        return builder;
    }

    public TypeSafeCustomerBuilder<TCustomerSet, IEmailSet> WithEmail(string email)
    {
        var builder = new TypeSafeCustomerBuilder<TCustomerSet, IEmailSet>
        {
            _firstName = this._firstName,
            _lastName = this._lastName,
            _email = email
        };
        return builder;
    }

    // Build only available when all required fields are set
    public Result<Customer> Build()
        where TCustomerSet : ICustomerSet
        where TEmailSet : IEmailSet
    {
        return Customer.Create(_firstName!, _lastName!, _email!);
    }
}

[TestClass]
public class TypeSafeBuilderTests
{
    [TestMethod]
    public void TypeSafeBuilder_WithAllRequiredData_Compiles()
    {
        // Arrange & Act
        var builder = TypeSafeCustomerBuilder<object, object>.Create()
            .WithName("John", "Doe")
            .WithEmail("john@example.com");

        var result = builder.Build();

        // Assert
        result.IsSuccess.Should().BeTrue();
    }

    // Note: The following would not compile:
    // var builder = TypeSafeCustomerBuilder<object, object>.Create();
    // builder.Build(); // ❌ Compile error: constraints not satisfied
}
```

## Testing Builder with Collections

```csharp
[TestClass]
public class BuilderCollectionTests
{
    [TestMethod]
    public void AddLine_AccumulatesLines()
    {
        // Arrange
        var line1 = CreateOrderLine(productId: ProductId.New());
        var line2 = CreateOrderLine(productId: ProductId.New());
        var line3 = CreateOrderLine(productId: ProductId.New());

        var builder = new OrderBuilder()
            .WithCustomer(CustomerId.New())
            .WithTotal(Money.USD(300m))
            .AddLine(line1)
            .AddLine(line2)
            .AddLine(line3);

        // Act
        var result = builder.Build();

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Lines.Should().HaveCount(3);
        result.Value.Lines.Should().Contain(line1);
        result.Value.Lines.Should().Contain(line2);
        result.Value.Lines.Should().Contain(line3);
    }

    [TestMethod]
    public void AddLines_AddMultipleAtOnce()
    {
        // Arrange
        var lines = new[]
        {
            CreateOrderLine(),
            CreateOrderLine(),
            CreateOrderLine()
        };

        var builder = new OrderBuilder()
            .WithCustomer(CustomerId.New())
            .WithTotal(Money.USD(300m))
            .AddLines(lines);

        // Act
        var result = builder.Build();

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Lines.Should().HaveCount(3);
    }
}
```

## Testing Builder Defaults and Overrides

```csharp
public class CustomerBuilder
{
    private string _firstName = "John";
    private string _lastName = "Doe";
    private string _email = "john.doe@example.com";
    private DateTime _dateOfBirth = DateTime.Now.AddYears(-30);

    public CustomerBuilder WithFirstName(string firstName)
    {
        _firstName = firstName;
        return this;
    }

    public CustomerBuilder WithLastName(string lastName)
    {
        _lastName = lastName;
        return this;
    }

    public CustomerBuilder WithEmail(string email)
    {
        _email = email;
        return this;
    }

    public CustomerBuilder WithDateOfBirth(DateTime dateOfBirth)
    {
        _dateOfBirth = dateOfBirth;
        return this;
    }

    public Result<Customer> Build() =>
        Customer.Create(_firstName, _lastName, _email, _dateOfBirth);
}

[TestClass]
public class BuilderDefaultsTests
{
    [TestMethod]
    public void Build_WithoutSetters_UsesDefaults()
    {
        // Arrange
        var builder = new CustomerBuilder();

        // Act
        var result = builder.Build();

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.FirstName.Should().Be("John");
        result.Value.LastName.Should().Be("Doe");
        result.Value.Email.Should().Be("john.doe@example.com");
    }

    [TestMethod]
    public void Build_OverridesDefaults()
    {
        // Arrange
        var builder = new CustomerBuilder()
            .WithFirstName("Jane")
            .WithEmail("jane@example.com");

        // Act
        var result = builder.Build();

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.FirstName.Should().Be("Jane");
        result.Value.LastName.Should().Be("Doe"); // Default
        result.Value.Email.Should().Be("jane@example.com");
    }

    [TestMethod]
    public void Build_AllowsOverridingMultipleTimes()
    {
        // Arrange
        var builder = new CustomerBuilder()
            .WithFirstName("Jane")
            .WithFirstName("Jack"); // Override again

        // Act
        var result = builder.Build();

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.FirstName.Should().Be("Jack");
    }
}
```

## Testing Builder Immutability

```csharp
[TestClass]
public class BuilderImmutabilityTests
{
    [TestMethod]
    public void Builder_ModificationsCreateNewInstances()
    {
        // Arrange
        var builder1 = new ImmutableOrderBuilder()
            .WithCustomer(CustomerId.New());

        // Act
        var builder2 = builder1.WithTotal(Money.USD(100m));

        // Assert
        builder1.Should().NotBeSameAs(builder2);
    }

    [TestMethod]
    public void Builder_OriginalUnchangedAfterModification()
    {
        // Arrange
        var customerId1 = CustomerId.New();
        var customerId2 = CustomerId.New();

        var builder1 = new ImmutableOrderBuilder()
            .WithCustomer(customerId1);

        // Act
        var builder2 = builder1.WithCustomer(customerId2);

        // Assert
        builder1.Build().Value.CustomerId.Should().Be(customerId1);
        builder2.Build().Value.CustomerId.Should().Be(customerId2);
    }
}
```

## Testing Nested Builders

```csharp
public class AddressBuilder
{
    private string _street = "";
    private string _city = "";
    private string _postalCode = "";
    private string _country = "";

    public AddressBuilder WithStreet(string street)
    {
        _street = street;
        return this;
    }

    public AddressBuilder WithCity(string city)
    {
        _city = city;
        return this;
    }

    public AddressBuilder WithPostalCode(string postalCode)
    {
        _postalCode = postalCode;
        return this;
    }

    public AddressBuilder WithCountry(string country)
    {
        _country = country;
        return this;
    }

    public Result<Address> Build() =>
        Address.Create(_street, _city, _postalCode, _country);
}

public class EnhancedCustomerBuilder
{
    private CustomerBuilder _customerBuilder = new();
    private AddressBuilder? _addressBuilder;

    public EnhancedCustomerBuilder WithCustomer(Action<CustomerBuilder> configure)
    {
        configure(_customerBuilder);
        return this;
    }

    public EnhancedCustomerBuilder WithAddress(Action<AddressBuilder> configure)
    {
        _addressBuilder = new AddressBuilder();
        configure(_addressBuilder);
        return this;
    }

    public Result<EnhancedCustomer> Build()
    {
        var customerResult = _customerBuilder.Build();
        if (customerResult.IsFailure)
            return Result.Failure<EnhancedCustomer>(customerResult.Error);

        Address? address = null;
        if (_addressBuilder is not null)
        {
            var addressResult = _addressBuilder.Build();
            if (addressResult.IsFailure)
                return Result.Failure<EnhancedCustomer>(addressResult.Error);
            address = addressResult.Value;
        }

        return EnhancedCustomer.Create(customerResult.Value, address);
    }
}

[TestClass]
public class NestedBuilderTests
{
    [TestMethod]
    public void NestedBuilder_ConfiguresChildBuilders()
    {
        // Arrange
        var builder = new EnhancedCustomerBuilder()
            .WithCustomer(c => c
                .WithFirstName("John")
                .WithLastName("Doe"))
            .WithAddress(a => a
                .WithStreet("123 Main St")
                .WithCity("Springfield"));

        // Act
        var result = builder.Build();

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Customer.FirstName.Should().Be("John");
        result.Value.Address.Should().NotBeNull();
        result.Value.Address!.Street.Should().Be("123 Main St");
    }
}
```

## Why It's Important

1. **Complexity Management**: Simplifies complex object creation
2. **Fluent API**: Expressive and readable construction
3. **Validation**: Centralized build validation
4. **Testability**: Easy to configure test objects
5. **Type Safety**: Compile-time safety with type-state pattern

## Guidelines

**Testing Builders**
- Test fluent interface returns self
- Test required vs optional properties
- Test build validation
- Test builder reusability
- Test defaults and overrides
- Test collection accumulation

**Validation Testing**
- Test Build() with missing required fields
- Test Build() with all required fields
- Test validation error messages
- Test optional field defaults

**Common Patterns**
- `.Should().BeSameAs(builder)` - fluent interface
- `.IsFailure.Should().BeTrue()` - validation
- `.Build().Value.Property.Should().Be()` - property verification

## Common Pitfalls

❌ **Not testing required fields**
```csharp
// Incomplete - only tests success path
builder.WithCustomer(id).WithTotal(total).Build()
    .IsSuccess.Should().BeTrue();
```

✅ **Test all required fields**
```csharp
// Complete
builder.WithTotal(total).Build().IsFailure.Should().BeTrue();
builder.WithCustomer(id).Build().IsFailure.Should().BeTrue();
builder.WithCustomer(id).WithTotal(total).Build().IsSuccess.Should().BeTrue();
```

❌ **Not testing fluent interface**
```csharp
// Missing fluent interface verification
builder.WithCustomer(id);
builder.WithTotal(total);
```

✅ **Verify fluent chaining**
```csharp
// Complete
builder.WithCustomer(id).Should().BeSameAs(builder);
builder.WithCustomer(id).WithTotal(total); // Chaining works
```

## See Also

- [Testing Factory Methods](./testing-factory-methods.md) — factory testing
- [Testing Smart Constructors](./testing-smart-constructors.md) — validated construction
- [Builder Pattern](../patterns/type-safe-builder.md) — builder implementation
- [Fluent Test Builders](./fluent-test-builders.md) — test data builders
