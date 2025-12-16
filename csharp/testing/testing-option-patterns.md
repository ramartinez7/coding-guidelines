# Testing Option Patterns

> Verify Option<T> monadic handling of optional values, Some/None cases, and mapping operations using FluentAssertions.

## Problem

Option types provide explicit handling of optional values without null. Tests must verify Some/None states, mapping operations, and proper option composition.

## Pattern

Test Option<T> for Some values, None states, mapping, binding, and option-specific operations using FluentAssertions.

## Example

### ❌ Before - Nullable Testing

```csharp
[TestMethod]
public void FindCustomer_ReturnsNullWhenNotFound()
{
    var customer = repository.Find(customerId);
    Assert.IsNull(customer);
    // ❌ Unclear intent with null
}
```

### ✅ After - Option Pattern Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

public record Option<T>
{
    public bool IsSome { get; }
    public bool IsNone => !IsSome;
    private readonly T _value;

    public T Value
    {
        get
        {
            if (IsNone)
                throw new InvalidOperationException("Cannot access value of None");
            return _value;
        }
    }

    private Option(T value, bool isSome)
    {
        _value = value;
        IsSome = isSome;
    }

    public static Option<T> Some(T value) => new(value, true);
    public static Option<T> None() => new(default!, false);

    public Option<TNew> Map<TNew>(Func<T, TNew> mapper) =>
        IsSome
            ? Option<TNew>.Some(mapper(Value))
            : Option<TNew>.None();

    public Option<TNew> Bind<TNew>(Func<T, Option<TNew>> binder) =>
        IsSome ? binder(Value) : Option<TNew>.None();

    public T GetValueOrDefault(T defaultValue) =>
        IsSome ? Value : defaultValue;
}

[TestClass]
public class OptionPatternTests
{
    [TestMethod]
    public void Some_CreatesOptionWithValue()
    {
        // Act
        var option = Option<int>.Some(42);

        // Assert
        option.IsSome.Should().BeTrue();
        option.IsNone.Should().BeFalse();
        option.Value.Should().Be(42);
    }

    [TestMethod]
    public void None_CreatesOptionWithoutValue()
    {
        // Act
        var option = Option<int>.None();

        // Assert
        option.IsNone.Should().BeTrue();
        option.IsSome.Should().BeFalse();
    }

    [TestMethod]
    public void Value_OnNone_ThrowsInvalidOperationException()
    {
        // Arrange
        var option = Option<int>.None();

        // Act
        Action act = () => { var _ = option.Value; };

        // Assert
        act.Should().Throw<InvalidOperationException>()
            .WithMessage("*Cannot access value*");
    }
}
```

## Testing Option Creation from Nullable

```csharp
public static class OptionExtensions
{
    public static Option<T> ToOption<T>(this T? value) where T : class =>
        value is not null
            ? Option<T>.Some(value)
            : Option<T>.None();

    public static Option<T> ToOption<T>(this T? value) where T : struct =>
        value.HasValue
            ? Option<T>.Some(value.Value)
            : Option<T>.None();
}

[TestClass]
public class OptionCreationTests
{
    [TestMethod]
    public void ToOption_WithNonNullValue_ReturnsSome()
    {
        // Arrange
        string value = "hello";

        // Act
        var option = value.ToOption();

        // Assert
        option.IsSome.Should().BeTrue();
        option.Value.Should().Be("hello");
    }

    [TestMethod]
    public void ToOption_WithNullValue_ReturnsNone()
    {
        // Arrange
        string? value = null;

        // Act
        var option = value.ToOption();

        // Assert
        option.IsNone.Should().BeTrue();
    }

    [TestMethod]
    public void ToOption_WithNullableInt_ReturnsSome()
    {
        // Arrange
        int? value = 42;

        // Act
        var option = value.ToOption();

        // Assert
        option.IsSome.Should().BeTrue();
        option.Value.Should().Be(42);
    }

    [TestMethod]
    public void ToOption_WithNullNullableInt_ReturnsNone()
    {
        // Arrange
        int? value = null;

        // Act
        var option = value.ToOption();

        // Assert
        option.IsNone.Should().BeTrue();
    }
}
```

## Testing Option Mapping

```csharp
[TestClass]
public class OptionMappingTests
{
    [TestMethod]
    public void Map_OnSome_TransformsValue()
    {
        // Arrange
        var option = Option<int>.Some(42);

        // Act
        var mapped = option.Map(x => x.ToString());

        // Assert
        mapped.IsSome.Should().BeTrue();
        mapped.Value.Should().Be("42");
    }

    [TestMethod]
    public void Map_OnNone_RemainsNone()
    {
        // Arrange
        var option = Option<int>.None();

        // Act
        var mapped = option.Map(x => x.ToString());

        // Assert
        mapped.IsNone.Should().BeTrue();
    }

    [TestMethod]
    public void Map_ChainedMappings_TransformsSuccessfully()
    {
        // Arrange
        var option = Option<int>.Some(10);

        // Act
        var final = option
            .Map(x => x * 2)
            .Map(x => x + 5)
            .Map(x => x.ToString());

        // Assert
        final.IsSome.Should().BeTrue();
        final.Value.Should().Be("25");
    }
}
```

## Testing Option Binding

```csharp
public class CustomerRepository
{
    public Option<Customer> FindById(CustomerId id) =>
        // Simulate lookup
        id.Value.ToString().StartsWith("valid")
            ? Option<Customer>.Some(new Customer(id, "John", "Doe"))
            : Option<Customer>.None();
}

[TestClass]
public class OptionBindingTests
{
    [TestMethod]
    public void Bind_OnSome_ExecutesBinder()
    {
        // Arrange
        var customerId = CustomerId.Parse("valid-123");
        var repository = new CustomerRepository();
        var option = Option<CustomerId>.Some(customerId);

        // Act
        var bound = option.Bind(id => repository.FindById(id));

        // Assert
        bound.IsSome.Should().BeTrue();
        bound.Value.FirstName.Should().Be("John");
    }

    [TestMethod]
    public void Bind_OnNone_RemainsNone()
    {
        // Arrange
        var repository = new CustomerRepository();
        var option = Option<CustomerId>.None();

        // Act
        var bound = option.Bind(id => repository.FindById(id));

        // Assert
        bound.IsNone.Should().BeTrue();
    }

    [TestMethod]
    public void Bind_WithBinderReturningNone_ReturnsNone()
    {
        // Arrange
        var customerId = CustomerId.Parse("invalid-123");
        var repository = new CustomerRepository();
        var option = Option<CustomerId>.Some(customerId);

        // Act
        var bound = option.Bind(id => repository.FindById(id));

        // Assert
        bound.IsNone.Should().BeTrue();
    }

    [TestMethod]
    public void Bind_ChainedBindings_ComposesCorrectly()
    {
        // Arrange
        var customerId = CustomerId.Parse("valid-123");
        var option = Option<CustomerId>.Some(customerId);

        // Act
        var final = option
            .Bind(id => FindCustomer(id))
            .Bind(customer => GetPrimaryEmail(customer));

        // Assert - Depends on implementation
        final.IsSome.Should().BeTrue();
    }

    private Option<Customer> FindCustomer(CustomerId id) =>
        Option<Customer>.Some(new Customer(id, "John", "Doe"));

    private Option<Email> GetPrimaryEmail(Customer customer) =>
        Option<Email>.Some(Email.Create("john@example.com").Value);
}
```

## Testing GetValueOrDefault

```csharp
[TestClass]
public class OptionGetValueOrDefaultTests
{
    [TestMethod]
    public void GetValueOrDefault_OnSome_ReturnsValue()
    {
        // Arrange
        var option = Option<int>.Some(42);

        // Act
        var value = option.GetValueOrDefault(0);

        // Assert
        value.Should().Be(42);
    }

    [TestMethod]
    public void GetValueOrDefault_OnNone_ReturnsDefault()
    {
        // Arrange
        var option = Option<int>.None();

        // Act
        var value = option.GetValueOrDefault(99);

        // Assert
        value.Should().Be(99);
    }

    [TestMethod]
    public void GetValueOrDefault_WithComplexDefault_Works()
    {
        // Arrange
        var option = Option<Customer>.None();
        var defaultCustomer = new Customer(CustomerId.New(), "Default", "User");

        // Act
        var value = option.GetValueOrDefault(defaultCustomer);

        // Assert
        value.Should().BeSameAs(defaultCustomer);
    }
}
```

## Testing Option Match

```csharp
public static class OptionMatchExtensions
{
    public static TResult Match<T, TResult>(
        this Option<T> option,
        Func<T, TResult> onSome,
        Func<TResult> onNone)
    {
        return option.IsSome ? onSome(option.Value) : onNone();
    }
}

[TestClass]
public class OptionMatchTests
{
    [TestMethod]
    public void Match_OnSome_ExecutesSomeFunction()
    {
        // Arrange
        var option = Option<int>.Some(42);

        // Act
        var output = option.Match(
            onSome: value => $"Found: {value}",
            onNone: () => "Not found"
        );

        // Assert
        output.Should().Be("Found: 42");
    }

    [TestMethod]
    public void Match_OnNone_ExecutesNoneFunction()
    {
        // Arrange
        var option = Option<int>.None();

        // Act
        var output = option.Match(
            onSome: value => $"Found: {value}",
            onNone: () => "Not found"
        );

        // Assert
        output.Should().Be("Not found");
    }
}
```

## Testing Option in Domain Logic

```csharp
public class OrderService
{
    public Option<Discount> CalculateDiscount(Customer customer, Order order)
    {
        if (!customer.IsVIP && order.Total.Amount < 1000m)
            return Option<Discount>.None();

        var percentage = customer.IsVIP ? 0.15m : 0.10m;
        return Option<Discount>.Some(new Discount(percentage));
    }
}

[TestClass]
public class DomainOptionTests
{
    [TestMethod]
    public void CalculateDiscount_VIPCustomer_ReturnsSome()
    {
        // Arrange
        var service = new OrderService();
        var customer = CreateVIPCustomer();
        var order = CreateOrder(total: Money.USD(500m));

        // Act
        var discount = service.CalculateDiscount(customer, order);

        // Assert
        discount.IsSome.Should().BeTrue();
        discount.Value.Percentage.Should().Be(0.15m);
    }

    [TestMethod]
    public void CalculateDiscount_RegularCustomerSmallOrder_ReturnsNone()
    {
        // Arrange
        var service = new OrderService();
        var customer = CreateRegularCustomer();
        var order = CreateOrder(total: Money.USD(500m));

        // Act
        var discount = service.CalculateDiscount(customer, order);

        // Assert
        discount.IsNone.Should().BeTrue();
    }

    [TestMethod]
    public void CalculateDiscount_RegularCustomerLargeOrder_ReturnsSome()
    {
        // Arrange
        var service = new OrderService();
        var customer = CreateRegularCustomer();
        var order = CreateOrder(total: Money.USD(1500m));

        // Act
        var discount = service.CalculateDiscount(customer, order);

        // Assert
        discount.IsSome.Should().BeTrue();
        discount.Value.Percentage.Should().Be(0.10m);
    }
}
```

## Testing Option with LINQ

```csharp
public static class OptionLinqExtensions
{
    public static Option<T> Where<T>(this Option<T> option, Func<T, bool> predicate) =>
        option.IsSome && predicate(option.Value)
            ? option
            : Option<T>.None();

    public static Option<TResult> Select<T, TResult>(
        this Option<T> option,
        Func<T, TResult> selector) =>
        option.Map(selector);
}

[TestClass]
public class OptionLinqTests
{
    [TestMethod]
    public void Where_WithMatchingPredicate_ReturnsSome()
    {
        // Arrange
        var option = Option<int>.Some(42);

        // Act
        var filtered = option.Where(x => x > 40);

        // Assert
        filtered.IsSome.Should().BeTrue();
        filtered.Value.Should().Be(42);
    }

    [TestMethod]
    public void Where_WithNonMatchingPredicate_ReturnsNone()
    {
        // Arrange
        var option = Option<int>.Some(42);

        // Act
        var filtered = option.Where(x => x > 50);

        // Assert
        filtered.IsNone.Should().BeTrue();
    }

    [TestMethod]
    public void Select_TransformsValue()
    {
        // Arrange
        var option = Option<int>.Some(42);

        // Act
        var selected = option.Select(x => x.ToString());

        // Assert
        selected.IsSome.Should().BeTrue();
        selected.Value.Should().Be("42");
    }

    [TestMethod]
    public void LinqQuery_ComposesCorrectly()
    {
        // Arrange
        var option = Option<int>.Some(42);

        // Act
        var result = from value in option
                     where value > 40
                     select value.ToString();

        // Assert
        result.IsSome.Should().BeTrue();
        result.Value.Should().Be("42");
    }
}
```

## Why It's Important

1. **Explicit Optionality**: Make optional values explicit
2. **No Null**: Avoid null reference exceptions
3. **Composability**: Chain optional operations
4. **Type Safety**: Compiler enforces presence checks
5. **Clarity**: Clear intent with Some/None

## Guidelines

**Testing Options**
- Test Some with various values
- Test None state explicitly
- Test mapping transformations
- Test binding operations
- Test pattern matching

**Common Patterns**
- `IsSome.Should().BeTrue()`
- `IsNone.Should().BeTrue()`
- `Value.Should().Be(expected)`
- Test GetValueOrDefault
- Test Match expressions

**Option Assertions**
- Verify Some/None state
- Check value only when Some
- Test default value behavior
- Test transformation correctness

## Common Pitfalls

❌ **Accessing Value on None**
```csharp
// Dangerous
var option = Option<int>.None();
var value = option.Value; // Throws!
```

✅ **Check IsSome first**
```csharp
// Safe
var option = Option<int>.None();
if (option.IsSome)
{
    var value = option.Value;
}
```

❌ **Not testing None case**
```csharp
// Incomplete
var option = FindCustomer(id);
option.IsSome.Should().BeTrue();
```

✅ **Test both cases**
```csharp
// Complete
FindCustomer(validId).IsSome.Should().BeTrue();
FindCustomer(invalidId).IsNone.Should().BeTrue();
```

## See Also

- [Testing Result Patterns](./testing-result-patterns.md) — result monad
- [Testing Nullable Types](./testing-nullable-value-types.md) — nullable values
- [Option Monad](../patterns/option-monad.md) — option pattern
- [Nullability and Optionality](../patterns/nullability-optionality.md) — optional values
