# Property-Based Testing

> Test invariants across many generated inputs—let the framework discover edge cases you didn't think of.

## Problem

Example-based tests only cover cases you explicitly write. Edge cases and unusual combinations often go untested until they cause production bugs.

## Pattern

Instead of writing specific examples, define properties that should hold for all inputs. The testing framework generates hundreds of random inputs to find counter-examples.

## Example

### ❌ Before - Example-Based Testing

```csharp
[Fact]
public void Add_TwoPositiveNumbers_ReturnsPositive()
{
    Assert.True(Calculator.Add(5, 3) > 0);
}

[Fact]
public void Add_WithZero_ReturnsOther()
{
    Assert.Equal(5, Calculator.Add(5, 0));
    Assert.Equal(3, Calculator.Add(0, 3));
}

// What about negative numbers? Large numbers? Overflow?
```

### ✅ After - Property-Based Testing

```csharp
[Property]
public Property Add_IsCommutative(int a, int b)
{
    // For all integers a and b: a + b == b + a
    return (Calculator.Add(a, b) == Calculator.Add(b, a)).ToProperty();
}

[Property]
public Property Add_ZeroIsIdentity(int a)
{
    // For all integers a: a + 0 == a
    return (Calculator.Add(a, 0) == a).ToProperty();
}

[Property]
public Property Add_IsAssociative(int a, int b, int c)
{
    // For all integers: (a + b) + c == a + (b + c)
    var left = Calculator.Add(Calculator.Add(a, b), c);
    var right = Calculator.Add(a, Calculator.Add(b, c));
    return (left == right).ToProperty();
}
```

## Using FsCheck in C#

Install: `dotnet add package FsCheck.Xunit`

### Basic Properties

```csharp
using FsCheck;
using FsCheck.Xunit;

public class MoneyTests
{
    [Property]
    public Property Add_IsCommutative(decimal a, decimal b)
    {
        var money1 = Money.USD(a);
        var money2 = Money.USD(b);
        
        var left = money1.Add(money2);
        var right = money2.Add(money1);
        
        return (left == right).ToProperty();
    }

    [Property]
    public Property Multiply_ByOne_ReturnsOriginal(decimal amount)
    {
        var money = Money.USD(amount);
        var result = money.Multiply(1);
        
        return (result == money).ToProperty();
    }

    [Property]
    public Property Negate_Twice_ReturnsOriginal(decimal amount)
    {
        var money = Money.USD(amount);
        var result = money.Negate().Negate();
        
        return (result == money).ToProperty();
    }
}
```

### Testing Domain Invariants

```csharp
public class EmailAddressTests
{
    [Property]
    public Property Create_WithValidEmail_RoundTripsCorrectly(string localPart, string domain)
    {
        // Only test with valid patterns
        var validLocalPart = string.IsNullOrWhiteSpace(localPart) 
            ? "test" 
            : new string(localPart.Take(64).Where(char.IsLetterOrDigit).ToArray());
            
        var validDomain = string.IsNullOrWhiteSpace(domain)
            ? "example.com"
            : new string(domain.Take(253).Where(c => char.IsLetterOrDigit(c) || c == '.').ToArray());

        if (string.IsNullOrEmpty(validLocalPart) || string.IsNullOrEmpty(validDomain))
            return true.ToProperty();

        var input = $"{validLocalPart}@{validDomain}";
        var result = EmailAddress.Create(input);
        
        return result.IsSuccess.ToProperty()
            .And(result.Value.ToString().Contains("@"))
            .Label($"Input: {input}");
    }
}
```

### Custom Generators

```csharp
public class CustomGenerators
{
    public static Arbitrary<Money> MoneyGenerator() =>
        Arb.From(
            from amount in Arb.Default.Decimal().Generator
            from currency in Gen.Elements(Currency.USD, Currency.EUR, Currency.GBP)
            where amount >= 0 && amount < 1000000
            select Money.Create(amount, currency)
        );

    public static Arbitrary<CustomerId> CustomerIdGenerator() =>
        Arb.From(
            from id in Arb.Default.Guid().Generator
            select CustomerId.From(id)
        );
}

[Properties(Arbitrary = new[] { typeof(CustomGenerators) })]
public class OrderTests
{
    [Property]
    public Property CreateOrder_WithValidMoney_Succeeds(Money amount)
    {
        var customerId = CustomerId.New();
        var result = Order.Create(customerId, amount);
        
        return result.IsSuccess.ToProperty();
    }
}
```

### Testing Serialization

```csharp
[Property]
public Property Serialize_Deserialize_RoundTrips(Order order)
{
    var json = JsonSerializer.Serialize(order);
    var deserialized = JsonSerializer.Deserialize<Order>(json);
    
    return (deserialized == order).ToProperty();
}

[Property]
public Property ToString_Parse_RoundTrips(Money money)
{
    var text = money.ToString();
    var parsed = Money.Parse(text);
    
    return (parsed == money).ToProperty();
}
```

## Properties to Test

### Algebraic Properties

```csharp
// Commutativity: a + b == b + a
[Property]
public Property Add_IsCommutative(int a, int b) =>
    (a + b == b + a).ToProperty();

// Associativity: (a + b) + c == a + (b + c)
[Property]
public Property Add_IsAssociative(int a, int b, int c) =>
    ((a + b) + c == a + (b + c)).ToProperty();

// Identity: a + 0 == a
[Property]
public Property Add_ZeroIsIdentity(int a) =>
    (a + 0 == a).ToProperty();

// Inverse: a + (-a) == 0
[Property]
public Property Add_Inverse(int a) =>
    (a + (-a) == 0).ToProperty();
```

### Idempotence

```csharp
[Property]
public Property Normalize_IsIdempotent(string input)
{
    var normalized1 = StringNormalizer.Normalize(input);
    var normalized2 = StringNormalizer.Normalize(normalized1);
    
    return (normalized1 == normalized2).ToProperty();
}

[Property]
public Property Trim_IsIdempotent(string input)
{
    var trimmed1 = input?.Trim();
    var trimmed2 = trimmed1?.Trim();
    
    return (trimmed1 == trimmed2).ToProperty();
}
```

### Invariants

```csharp
[Property]
public Property OrderTotal_AlwaysNonNegative(Order order)
{
    return (order.Total >= Money.USD(0)).ToProperty();
}

[Property]
public Property Collection_CountMatchesList(List<int> items)
{
    var collection = new CustomCollection<int>(items);
    
    return (collection.Count == items.Count).ToProperty();
}
```

### Relationships

```csharp
[Property]
public Property SortedList_IsOrderedAscending(List<int> items)
{
    var sorted = items.OrderBy(x => x).ToList();
    
    for (int i = 0; i < sorted.Count - 1; i++)
    {
        if (sorted[i] > sorted[i + 1])
            return false.ToProperty();
    }
    
    return true.ToProperty();
}
```

## Conditional Properties

```csharp
[Property]
public Property Divide_NonZero_DoesNotThrow(int a, int b)
{
    // Only test when b is not zero
    return (b != 0).ImpliesProperty(() =>
    {
        var result = Calculator.Divide(a, b);
        return true;
    });
}

[Property]
public Property CreateAge_ValidRange_Succeeds(int value)
{
    return (value >= 0 && value <= 150).ImpliesProperty(() =>
    {
        var result = Age.Create(value);
        return result.IsSuccess.ToProperty();
    });
}
```

## Shrinking

When a property fails, FsCheck automatically shrinks the input to find the minimal failing case:

```csharp
[Property]
public Property Sort_PreservesElements(List<int> items)
{
    var sorted = items.OrderBy(x => x).ToList();
    
    // If this fails with a 100-element list, FsCheck will shrink
    // to find the smallest list that fails
    return (sorted.Count == items.Count).ToProperty();
}

// Output when failing:
// Falsifiable, after 1 test (0 shrinks):
// Original: [5, 3, 8, 1, 9, ...]
// Shrunk:   [1]
```

## Guidelines

1. **Start simple**: Test basic algebraic properties first
2. **Use constraints**: Apply preconditions with `Implies`
3. **Custom generators**: Create domain-appropriate test data
4. **Label failures**: Use `.Label()` to add context
5. **Combine with examples**: Use property tests AND example tests

## Benefits

1. **Finds edge cases**: Discovers bugs you didn't think to test
2. **Documents invariants**: Properties describe system behavior
3. **Confidence**: Hundreds of tests from one property
4. **Regression prevention**: Random inputs catch future breaks
5. **Better design**: Forces thinking about invariants

## See Also

- [Testing Domain Invariants](./testing-domain-invariants.md) — verifying business rules
- [Testing Value Objects](./testing-value-objects.md) — equality and validation
- [Parameterized Tests](./parameterized-tests.md) — testing multiple examples
- [Testing Discriminated Unions](./testing-discriminated-unions.md) — exhaustive testing
