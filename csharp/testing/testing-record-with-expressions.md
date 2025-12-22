# Testing Record With-Expressions

> Test record with-expressions and cloning—verify immutable updates and shallow vs deep copying.

## Problem

Record with-expressions create modified copies. Tests must verify originals are unchanged and copies have expected modifications.

## Example

```csharp
public record Customer(string Name, string Email, Address Address);
public record Address(string Street, string City);

[Fact]
public void WithExpression_ModifiesProperty_OriginalUnchanged()
{
    var original = new Customer("John", "john@example.com", 
        new Address("Main St", "Boston"));
    
    var updated = original with { Name = "Jane" };
    
    original.Name.Should().Be("John");
    updated.Name.Should().Be("Jane");
    updated.Email.Should().Be(original.Email);
}

[Fact]
public void WithExpression_NestedRecord_ShallowCopy()
{
    var original = new Customer("John", "john@example.com",
        new Address("Main St", "Boston"));
    
    var updated = original with { Name = "Jane" };
    
    updated.Address.Should().BeSameAs(original.Address);
}
```

## Testing Deep Copying

```csharp
[Fact]
public void DeepCopy_NestedRecords_CopiesAllLevels()
{
    var original = new Customer("John", "john@example.com",
        new Address("Main St", "Boston"));
    
    var updated = original with 
    { 
        Address = original.Address with { City = "Cambridge" }
    };
    
    original.Address.City.Should().Be("Boston");
    updated.Address.City.Should().Be("Cambridge");
    updated.Address.Should().NotBeSameAs(original.Address);
}
```

## See Also

- [Testing Record Types](./testing-record-types.md)
- [Testing Value Objects](./testing-value-objects.md)
