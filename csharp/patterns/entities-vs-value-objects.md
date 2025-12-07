# Entities vs Value Objects (Identity vs Equality)

> Objects with mutable state and identity tracked over time—distinguish entities from value objects to model domain accurately.

## Problem

When all objects are treated the same way, it's unclear which objects have identity that matters (entities) and which are defined purely by their attributes (value objects). This leads to incorrect equality comparisons, inefficient change tracking, and confused domain models.

## Example

### ❌ Before

```csharp
public class Customer
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class Address
{
    public Guid Id { get; set; }  // Address has ID but why?
    public string Street { get; set; }
    public string City { get; set; }
    public string PostalCode { get; set; }
    public string Country { get; set; }
}

// Confusing equality: two addresses with same values but different IDs
var addr1 = new Address 
{ 
    Id = Guid.NewGuid(), 
    Street = "123 Main St", 
    City = "Boston", 
    PostalCode = "02101",
    Country = "USA"
};

var addr2 = new Address 
{ 
    Id = Guid.NewGuid(),  // Different ID
    Street = "123 Main St", 
    City = "Boston", 
    PostalCode = "02101",
    Country = "USA"
};

// Are these the same address? ReferenceEquals says no, but should they be equal?
Console.WriteLine(addr1 == addr2);  // False—but they represent the same location!
```

**Problems:**
- Everything has an ID, even concepts that don't need identity
- Value objects with IDs create confusion about equality
- Mutable value objects lead to cache bugs
- No clear distinction between entities and values

### ✅ After

```csharp
/// <summary>
/// Entity: Identity matters. A customer is the same customer even if their name changes.
/// </summary>
public sealed class Customer
{
    private readonly List<Address> _addresses = new();
    
    public CustomerId Id { get; }  // Identity
    public string Name { get; private set; }  // Can change
    public EmailAddress Email { get; private set; }  // Can change
    public DateTimeOffset CreatedAt { get; }  // Immutable timestamp
    
    public IReadOnlyList<Address> Addresses => _addresses.AsReadOnly();
    
    private Customer(CustomerId id, string name, EmailAddress email, DateTimeOffset createdAt)
    {
        Id = id;
        Name = name;
        Email = email;
        CreatedAt = createdAt;
    }
    
    public static Customer Create(string name, EmailAddress email)
    {
        return new Customer(
            CustomerId.New(),
            name,
            email,
            DateTimeOffset.UtcNow);
    }
    
    // Entities can change over time
    public void ChangeName(string newName)
    {
        if (string.IsNullOrWhiteSpace(newName))
            throw new ArgumentException("Name cannot be empty");
        
        Name = newName;
    }
    
    public void ChangeEmail(EmailAddress newEmail)
    {
        Email = newEmail;
    }
    
    // Identity-based equality
    public override bool Equals(object? obj)
    {
        return obj is Customer other && Id == other.Id;
    }
    
    public override int GetHashCode() => Id.GetHashCode();
}

/// <summary>
/// Value Object: No identity. An address is defined entirely by its attributes.
/// Two addresses with the same values are the same address.
/// </summary>
public sealed record Address(
    string Street,
    string City,
    string PostalCode,
    string Country)
{
    // Static factory enforces validation
    public static Result<Address, string> Create(
        string street,
        string city,
        string postalCode,
        string country)
    {
        if (string.IsNullOrWhiteSpace(street))
            return Result<Address, string>.Failure("Street is required");
        
        if (string.IsNullOrWhiteSpace(city))
            return Result<Address, string>.Failure("City is required");
        
        if (string.IsNullOrWhiteSpace(postalCode))
            return Result<Address, string>.Failure("Postal code is required");
        
        if (string.IsNullOrWhiteSpace(country))
            return Result<Address, string>.Failure("Country is required");
        
        return Result<Address, string>.Success(
            new Address(street, city, postalCode, country));
    }
    
    // Value objects are naturally equal by value (using record)
    // No need to override Equals/GetHashCode
}

// Usage: Clear distinction
var customer = Customer.Create("Alice", EmailAddress.Parse("alice@example.com"));

var address1 = Address.Create("123 Main St", "Boston", "02101", "USA").Value;
var address2 = Address.Create("123 Main St", "Boston", "02101", "USA").Value;

// Value objects: equal if values are equal
Console.WriteLine(address1 == address2);  // True—same location

// Entities: equal only if IDs match
var customer2 = Customer.Create("Alice", EmailAddress.Parse("alice@example.com"));
Console.WriteLine(customer == customer2);  // False—different customers
```

## Characteristics

### Entity Characteristics

| Characteristic | Description | Example |
|----------------|-------------|---------|
| **Has Identity** | Tracked by unique ID over lifetime | `Customer`, `Order`, `Product` |
| **Mutable** | State can change | Customer changes address |
| **Identity Equality** | Same ID = same entity | Customer with ID=42 is always that customer |
| **Lifecycle** | Created, modified, deleted | Order placed → shipped → delivered |
| **Continuity** | Identity persists through changes | Customer is same person after name change |

```csharp
// Entity example: Order
public sealed class Order
{
    public OrderId Id { get; }  // Identity never changes
    public OrderStatus Status { get; private set; }  // State changes
    public Money Total { get; private set; }  // State changes
    
    public void Ship()
    {
        // Same order, different state
        Status = OrderStatus.Shipped;
    }
}
```

### Value Object Characteristics

| Characteristic | Description | Example |
|----------------|-------------|---------|
| **No Identity** | Defined entirely by attributes | `Address`, `Money`, `DateRange` |
| **Immutable** | Cannot change after creation | Money amount never changes |
| **Value Equality** | Equal if all attributes equal | $100 == $100 |
| **No Lifecycle** | No concept of "same" over time | Address doesn't "change", you get new address |
| **Replaceable** | Swap one for another if equal | Replace old address with new one |

```csharp
// Value object example: Money
public sealed record Money(decimal Amount, Currency Currency)
{
    // Immutable: operations return new instance
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        
        return new Money(Amount + other.Amount, Currency);
    }
    
    // Value equality automatic with records
}
```

## When to Use Each

### Use Entity When:

- Identity matters more than attributes
- Object has a lifecycle (created, modified, deleted)
- State changes over time
- Need to track history of changes
- Different instances with same data are distinct

**Examples:**
- `Customer` — Alice today is the same Alice tomorrow
- `Order` — Order #12345 is tracked from placement to delivery
- `BankAccount` — Account identity persists, balance changes
- `User` — User identity independent of profile changes

### Use Value Object When:

- Identity doesn't matter, only attributes
- Immutable by nature
- Interchangeable with any equal instance
- No lifecycle or history
- Represents a measurement, quantity, or descriptor

**Examples:**
- `Address` — Two identical addresses are the same location
- `Money` — $100 is $100, no identity needed
- `DateRange` — Span from Jan 1 to Jan 31 has no identity
- `EmailAddress` — alice@example.com is always that address

## Composition

```csharp
// Entities can contain value objects
public sealed class Customer
{
    public CustomerId Id { get; }  // Entity ID (value object)
    public PersonName Name { get; private set; }  // Value object
    public EmailAddress Email { get; private set; }  // Value object
    public Address HomeAddress { get; private set; }  // Value object
}

// Value objects can contain other value objects
public sealed record PersonName(string FirstName, string LastName)
{
    public string FullName => $"{FirstName} {LastName}";
}

// Entities can contain collections of value objects
public sealed class ShoppingCart
{
    private readonly List<CartItem> _items = new();
    
    public CartId Id { get; }  // Entity
    public IReadOnlyList<CartItem> Items => _items.AsReadOnly();  // Value objects
}

public sealed record CartItem(ProductId Product, int Quantity, Money UnitPrice);
```

## Anti-Pattern: Value Object with Identity

```csharp
// ❌ Anti-pattern: Address with unnecessary ID
public class Address
{
    public Guid AddressId { get; set; }  // Why does a value need identity?
    public string Street { get; set; }
    public string City { get; set; }
}

// Problem: What does equality mean?
var addr1 = new Address { AddressId = Guid.NewGuid(), Street = "123 Main", City = "Boston" };
var addr2 = new Address { AddressId = Guid.NewGuid(), Street = "123 Main", City = "Boston" };

// Same location, but different IDs—are they equal or not?

// ✅ Solution: Remove identity if it's truly a value
public sealed record Address(string Street, string City)
{
    // Equal by value, no identity needed
}
```

## Anti-Pattern: Entity with Value Equality

```csharp
// ❌ Anti-pattern: Order compared by attributes
public class Order
{
    public OrderId Id { get; set; }
    public CustomerId CustomerId { get; set; }
    public Money Total { get; set; }
    
    // Wrong: Entity equality based on attributes
    public override bool Equals(object? obj)
    {
        return obj is Order other &&
               CustomerId == other.CustomerId &&
               Total == other.Total;
    }
}

// Problem: Two different orders can be "equal"
var order1 = new Order { Id = OrderId.New(), CustomerId = customerId, Total = Money.USD(100) };
var order2 = new Order { Id = OrderId.New(), CustomerId = customerId, Total = Money.USD(100) };

// Different orders, but Equals returns true!

// ✅ Solution: Entity equality based on ID only
public sealed class Order
{
    public OrderId Id { get; }
    
    public override bool Equals(object? obj)
    {
        return obj is Order other && Id == other.Id;
    }
}
```

## Persistence Considerations

```csharp
// Entities: Mapped to tables with primary keys
public class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.HasKey(c => c.Id);  // Entity ID is primary key
        
        // Value objects owned by entity
        builder.OwnsOne(c => c.Email);
        builder.OwnsOne(c => c.HomeAddress);
    }
}

// Value objects: Owned types or value converters
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        
        // Value object stored in same table
        builder.OwnsOne(o => o.Total, money =>
        {
            money.Property(m => m.Amount).HasColumnName("TotalAmount");
            money.Property(m => m.Currency).HasColumnName("Currency");
        });
    }
}
```

## Why It's a Problem

1. **Incorrect equality**: Comparing entities by value or values by identity
2. **Confused models**: Everything is mutable or everything has ID
3. **Performance issues**: Tracking value objects in change detection
4. **Domain inaccuracy**: Code doesn't reflect real-world concepts

## Symptoms

- Value objects with database IDs
- Entities compared by attributes instead of ID
- Mutable value objects causing cache bugs
- Confusion about when objects are "the same"

## Benefits

- **Clear semantics**: Identity vs value explicit in code
- **Correct equality**: Entities by ID, values by attributes
- **Better modeling**: Code reflects domain concepts accurately
- **Immutable values**: Value objects safe to share and cache

## See Also

- [Aggregate Roots](./aggregate-roots.md) — entities as consistency boundaries
- [Value Semantics](./value-semantics.md) — implementing value objects
- [Strongly Typed IDs](./strongly-typed-ids.md) — value objects for identity
- [Repository Pattern](./repository-pattern.md) — persisting entities
