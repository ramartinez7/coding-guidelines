# Testing Record Types

> Verify record type behavior—equality, immutability, with-expressions, and deconstruction.

## Problem

Record types in C# provide value-based equality and immutability by default, but these behaviors should still be tested to ensure they work correctly in your domain model.

## Pattern

Test that record types properly implement value equality, support immutable updates via `with` expressions, and allow proper deconstruction.

## Example

### ❌ Before - Incomplete Record Testing

```csharp
[TestMethod]
public void CreateCustomer_Works()
{
    var customer = new Customer("John", "Doe");
    Assert.IsNotNull(customer);
}
// ❌ Missing: equality, with-expressions, deconstruction
```

### ✅ After - Complete Record Testing

```csharp
using FluentAssertions;
using Microsoft.VisualStudio.TestTools.UnitTesting;

public record Customer(string FirstName, string LastName, string Email);

[TestClass]
public class CustomerRecordTests
{
    [TestMethod]
    public void TwoRecords_WithSameValues_AreEqual()
    {
        // Arrange
        var customer1 = new Customer("John", "Doe", "john@example.com");
        var customer2 = new Customer("John", "Doe", "john@example.com");

        // Act & Assert
        customer1.Should().Be(customer2);
        (customer1 == customer2).Should().BeTrue();
    }

    [TestMethod]
    public void TwoRecords_WithDifferentValues_AreNotEqual()
    {
        // Arrange
        var customer1 = new Customer("John", "Doe", "john@example.com");
        var customer2 = new Customer("Jane", "Doe", "jane@example.com");

        // Act & Assert
        customer1.Should().NotBe(customer2);
        (customer1 != customer2).Should().BeTrue();
    }

    [TestMethod]
    public void WithExpression_CreatesNewInstance_WithUpdatedProperty()
    {
        // Arrange
        var original = new Customer("John", "Doe", "john@example.com");

        // Act
        var updated = original with { Email = "newemail@example.com" };

        // Assert
        updated.Should().NotBeSameAs(original);
        updated.FirstName.Should().Be("John");
        updated.LastName.Should().Be("Doe");
        updated.Email.Should().Be("newemail@example.com");
    }

    [TestMethod]
    public void WithExpression_DoesNotMutateOriginal()
    {
        // Arrange
        var original = new Customer("John", "Doe", "john@example.com");
        var originalEmail = original.Email;

        // Act
        var updated = original with { Email = "newemail@example.com" };

        // Assert
        original.Email.Should().Be(originalEmail);
    }

    [TestMethod]
    public void Record_CanBeDeconstructed()
    {
        // Arrange
        var customer = new Customer("John", "Doe", "john@example.com");

        // Act
        var (firstName, lastName, email) = customer;

        // Assert
        firstName.Should().Be("John");
        lastName.Should().Be("Doe");
        email.Should().Be("john@example.com");
    }

    [TestMethod]
    public void Records_WithSameValues_HaveSameHashCode()
    {
        // Arrange
        var customer1 = new Customer("John", "Doe", "john@example.com");
        var customer2 = new Customer("John", "Doe", "john@example.com");

        // Act
        var hash1 = customer1.GetHashCode();
        var hash2 = customer2.GetHashCode();

        // Assert
        hash1.Should().Be(hash2);
    }
}
```

## Testing Record Inheritance

```csharp
public record Person(string FirstName, string LastName);
public record Employee(string FirstName, string LastName, string EmployeeId) : Person(FirstName, LastName);

[TestClass]
public class RecordInheritanceTests
{
    [TestMethod]
    public void DerivedRecord_IsNotEqualToBase_EvenWithSameBaseProperties()
    {
        // Arrange
        var person = new Person("John", "Doe");
        var employee = new Employee("John", "Doe", "EMP001");

        // Act & Assert
        person.Should().NotBe(employee);
        employee.Should().NotBe(person);
    }

    [TestMethod]
    public void WithExpression_OnDerivedRecord_PreservesDerivedType()
    {
        // Arrange
        var employee = new Employee("John", "Doe", "EMP001");

        // Act
        var updated = employee with { FirstName = "Jane" };

        // Assert
        updated.Should().BeOfType<Employee>();
        updated.EmployeeId.Should().Be("EMP001");
        updated.FirstName.Should().Be("Jane");
    }
}
```

## Testing Record ToString

```csharp
[TestClass]
public class RecordToStringTests
{
    [TestMethod]
    public void ToString_ReturnsStructuredRepresentation()
    {
        // Arrange
        var customer = new Customer("John", "Doe", "john@example.com");

        // Act
        var result = customer.ToString();

        // Assert
        result.Should().Contain("John");
        result.Should().Contain("Doe");
        result.Should().Contain("john@example.com");
    }
}
```

## Why It's Important

1. **Value Semantics**: Records are designed for value-based equality—verify it works
2. **Immutability**: `with` expressions create new instances—ensure originals aren't mutated
3. **Hash Code Consistency**: Equal records must have equal hash codes for collections
4. **Deconstruction**: Pattern matching and tuple deconstruction are key record features
5. **Type Safety**: Derived records have distinct types—verify type relationships

## Guidelines

**What to Test**
- Equality of records with same values
- Inequality of records with different values
- Hash code consistency for equal records
- Immutability via `with` expressions
- Deconstruction into components
- ToString representation (when relied upon)
- Record inheritance relationships

**FluentAssertions for Records**
- Use `.Should().Be()` for value equality
- Use `.Should().NotBeSameAs()` to verify different instances
- Use `.Should().BeOfType<T>()` for derived record types
- Use `.Should().Contain()` for ToString verification

## Common Pitfalls

❌ **Comparing records with `ReferenceEquals`**
```csharp
// Bad - records use value equality
ReferenceEquals(customer1, customer2).Should().BeTrue();
```

✅ **Use value equality**
```csharp
// Good - test value equality
customer1.Should().Be(customer2);
```

❌ **Testing only constructor**
```csharp
// Incomplete
var customer = new Customer("John", "Doe", "john@example.com");
customer.Should().NotBeNull();
```

✅ **Test all record semantics**
```csharp
// Complete - test equality, immutability, deconstruction
customer1.Should().Be(customer2);
var updated = customer with { Email = "new@example.com" };
updated.Should().NotBeSameAs(customer);
```

## See Also

- [Testing Value Objects](./testing-value-objects.md) — value semantics testing
- [Testing Equality and Comparison](./testing-equality-comparison.md) — equality patterns
- [Value Semantics](../patterns/value-semantics.md) — value type philosophy
- [Record Equality](../patterns/record-equality.md) — record equality patterns
