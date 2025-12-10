# Value Object Composition (Building Complex Values from Simple Ones)

> Primitive values scattered throughout code—compose value objects from other value objects to model rich domain concepts with guaranteed consistency.

## Problem

Complex domain concepts are represented as collections of primitive fields or simple value objects, with validation and composition logic scattered across the codebase. There's no single type that encapsulates the full concept with its invariants.

## Example

### ❌ Before

```csharp
public class User
{
    public string Street { get; set; }
    public string City { get; set; }
    public string State { get; set; }
    public string ZipCode { get; set; }
    public string Country { get; set; }
    
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string MiddleName { get; set; }
    
    public string Email { get; set; }
    public string PhoneCountryCode { get; set; }
    public string PhoneNumber { get; set; }
}

// Validation scattered everywhere
public bool IsValidUser(User user)
{
    if (string.IsNullOrEmpty(user.Street)) return false;
    if (string.IsNullOrEmpty(user.City)) return false;
    if (!Regex.IsMatch(user.ZipCode, @"^\d{5}$")) return false;
    if (!Regex.IsMatch(user.Email, @"^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$")) return false;
    // ... more validation
    return true;
}
```

**Problems:**
- Validation logic scattered across methods
- No guarantee address fields are consistent
- Can't reuse address or name concepts
- Difficult to test individual concepts

### ✅ After

```csharp
// Base value objects
public sealed record EmailAddress
{
    public string Value { get; }
    
    private EmailAddress(string value) => Value = value;
    
    public static Result<EmailAddress, string> Create(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            return Result<EmailAddress, string>.Failure("Email cannot be empty");
        
        if (!Regex.IsMatch(email, @"^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$"))
            return Result<EmailAddress, string>.Failure("Invalid email format");
        
        return Result<EmailAddress, string>.Success(new EmailAddress(email));
    }
}

public sealed record PostalCode
{
    public string Value { get; }
    
    private PostalCode(string value) => Value = value;
    
    public static Result<PostalCode, string> Create(string code, Country country)
    {
        if (string.IsNullOrWhiteSpace(code))
            return Result<PostalCode, string>.Failure("Postal code cannot be empty");
        
        var isValid = country.Value switch
        {
            "US" => Regex.IsMatch(code, @"^\d{5}(-\d{4})?$"),
            "CA" => Regex.IsMatch(code, @"^[A-Z]\d[A-Z] \d[A-Z]\d$"),
            "UK" => Regex.IsMatch(code, @"^[A-Z]{1,2}\d{1,2} \d[A-Z]{2}$"),
            _ => true  // Accept any format for other countries
        };
        
        if (!isValid)
            return Result<PostalCode, string>.Failure($"Invalid postal code for {country.Value}");
        
        return Result<PostalCode, string>.Success(new PostalCode(code));
    }
}

public sealed record Country
{
    public string Value { get; }
    
    private Country(string value) => Value = value;
    
    public static Result<Country, string> Create(string code)
    {
        if (string.IsNullOrWhiteSpace(code))
            return Result<Country, string>.Failure("Country code cannot be empty");
        
        if (code.Length != 2)
            return Result<Country, string>.Failure("Country code must be 2 characters");
        
        return Result<Country, string>.Success(new Country(code.ToUpperInvariant()));
    }
}

// Composed value object: Address
public sealed record Address
{
    public string Street { get; }
    public string City { get; }
    public string State { get; }
    public PostalCode PostalCode { get; }
    public Country Country { get; }
    
    private Address(
        string street,
        string city,
        string state,
        PostalCode postalCode,
        Country country)
    {
        Street = street;
        City = city;
        State = state;
        PostalCode = postalCode;
        Country = country;
    }
    
    public static Result<Address, string> Create(
        string street,
        string city,
        string state,
        string postalCode,
        string country)
    {
        if (string.IsNullOrWhiteSpace(street))
            return Result<Address, string>.Failure("Street cannot be empty");
        
        if (string.IsNullOrWhiteSpace(city))
            return Result<Address, string>.Failure("City cannot be empty");
        
        var countryResult = Country.Create(country);
        if (!countryResult.IsSuccess)
            return Result<Address, string>.Failure(countryResult.Error);
        
        var postalCodeResult = PostalCode.Create(postalCode, countryResult.Value);
        if (!postalCodeResult.IsSuccess)
            return Result<Address, string>.Failure(postalCodeResult.Error);
        
        return Result<Address, string>.Success(new Address(
            street,
            city,
            state,
            postalCodeResult.Value,
            countryResult.Value));
    }
    
    public string ToFormattedString() => 
        $"{Street}\n{City}, {State} {PostalCode.Value}\n{Country.Value}";
}

// Composed value object: PersonName
public sealed record PersonName
{
    public string FirstName { get; }
    public string LastName { get; }
    public Option<string> MiddleName { get; }
    
    private PersonName(string firstName, string lastName, Option<string> middleName)
    {
        FirstName = firstName;
        LastName = lastName;
        MiddleName = middleName;
    }
    
    public static Result<PersonName, string> Create(
        string firstName,
        string lastName,
        Option<string> middleName = default)
    {
        if (string.IsNullOrWhiteSpace(firstName))
            return Result<PersonName, string>.Failure("First name cannot be empty");
        
        if (string.IsNullOrWhiteSpace(lastName))
            return Result<PersonName, string>.Failure("Last name cannot be empty");
        
        if (firstName.Length > 50)
            return Result<PersonName, string>.Failure("First name too long");
        
        if (lastName.Length > 50)
            return Result<PersonName, string>.Failure("Last name too long");
        
        return Result<PersonName, string>.Success(
            new PersonName(firstName.Trim(), lastName.Trim(), middleName));
    }
    
    public string FullName => MiddleName.Match(
        onSome: middle => $"{FirstName} {middle} {LastName}",
        onNone: () => $"{FirstName} {LastName}");
    
    public string FormalName => $"{LastName}, {FirstName}";
}

// Composed value object: PhoneNumber
public sealed record PhoneNumber
{
    public string CountryCode { get; }
    public string Number { get; }
    
    private PhoneNumber(string countryCode, string number)
    {
        CountryCode = countryCode;
        Number = number;
    }
    
    public static Result<PhoneNumber, string> Create(string countryCode, string number)
    {
        if (string.IsNullOrWhiteSpace(countryCode))
            return Result<PhoneNumber, string>.Failure("Country code cannot be empty");
        
        if (!countryCode.StartsWith("+"))
            return Result<PhoneNumber, string>.Failure("Country code must start with +");
        
        if (string.IsNullOrWhiteSpace(number))
            return Result<PhoneNumber, string>.Failure("Phone number cannot be empty");
        
        var digitsOnly = Regex.Replace(number, @"[^\d]", "");
        
        if (digitsOnly.Length < 7 || digitsOnly.Length > 15)
            return Result<PhoneNumber, string>.Failure("Invalid phone number length");
        
        return Result<PhoneNumber, string>.Success(
            new PhoneNumber(countryCode, digitsOnly));
    }
    
    public string Formatted => $"{CountryCode} {Number}";
}

// Top-level composed value object: ContactInfo
public sealed record ContactInfo
{
    public EmailAddress Email { get; }
    public Option<PhoneNumber> Phone { get; }
    public Address Address { get; }
    
    private ContactInfo(
        EmailAddress email,
        Option<PhoneNumber> phone,
        Address address)
    {
        Email = email;
        Phone = phone;
        Address = address;
    }
    
    public static Result<ContactInfo, string> Create(
        string email,
        Option<(string countryCode, string number)> phone,
        string street,
        string city,
        string state,
        string postalCode,
        string country)
    {
        var emailResult = EmailAddress.Create(email);
        if (!emailResult.IsSuccess)
            return Result<ContactInfo, string>.Failure(emailResult.Error);
        
        var addressResult = Address.Create(street, city, state, postalCode, country);
        if (!addressResult.IsSuccess)
            return Result<ContactInfo, string>.Failure(addressResult.Error);
        
        Option<PhoneNumber> phoneNumber = Option<PhoneNumber>.None();
        
        if (phone.IsSome)
        {
            var (countryCode, number) = phone.Value;
            var phoneResult = PhoneNumber.Create(countryCode, number);
            
            if (!phoneResult.IsSuccess)
                return Result<ContactInfo, string>.Failure(phoneResult.Error);
            
            phoneNumber = Option<PhoneNumber>.Some(phoneResult.Value);
        }
        
        return Result<ContactInfo, string>.Success(new ContactInfo(
            emailResult.Value,
            phoneNumber,
            addressResult.Value));
    }
}

// Entity using composed value objects
public sealed class User
{
    public UserId Id { get; }
    public PersonName Name { get; private set; }
    public ContactInfo ContactInfo { get; private set; }
    
    private User(UserId id, PersonName name, ContactInfo contactInfo)
    {
        Id = id;
        Name = name;
        ContactInfo = contactInfo;
    }
    
    public static Result<User, string> Create(
        UserId id,
        PersonName name,
        ContactInfo contactInfo)
    {
        return Result<User, string>.Success(new User(id, name, contactInfo));
    }
    
    public void UpdateContactInfo(ContactInfo newContactInfo)
    {
        ContactInfo = newContactInfo;
    }
}

// Usage: Compose from bottom up
var emailResult = EmailAddress.Create("john.doe@example.com");
var addressResult = Address.Create("123 Main St", "Springfield", "IL", "62701", "US");
var phoneResult = PhoneNumber.Create("+1", "2175551234");
var nameResult = PersonName.Create("John", "Doe", Option<string>.Some("Michael"));

// Compose contact info from validated parts
var contactInfoResult = emailResult
    .Bind(email => addressResult
        .Bind(address => phoneResult
            .Map(phone => new ContactInfo(
                email,
                Option<PhoneNumber>.Some(phone),
                address))));
```

## Composition Patterns

```csharp
// Pattern 1: Required composition (all parts required)
public sealed record Money
{
    public decimal Amount { get; }
    public Currency Currency { get; }
    
    private Money(decimal amount, Currency currency)
    {
        Amount = amount;
        Currency = currency;
    }
    
    public static Result<Money, string> Create(decimal amount, string currencyCode)
    {
        var currencyResult = Currency.Create(currencyCode);
        if (!currencyResult.IsSuccess)
            return Result<Money, string>.Failure(currencyResult.Error);
        
        if (amount < 0)
            return Result<Money, string>.Failure("Amount cannot be negative");
        
        return Result<Money, string>.Success(
            new Money(amount, currencyResult.Value));
    }
}

// Pattern 2: Optional composition (some parts optional)
public sealed record UserProfile
{
    public PersonName Name { get; }
    public Option<Avatar> Avatar { get; }
    public Option<Biography> Bio { get; }
    
    public static UserProfile Create(
        PersonName name,
        Option<Avatar> avatar,
        Option<Biography> bio) =>
        new UserProfile(name, avatar, bio);
}

// Pattern 3: Collection composition
public sealed record ShoppingCart
{
    public CartId Id { get; }
    public CustomerId CustomerId { get; }
    private readonly List<CartItem> _items;
    public IReadOnlyList<CartItem> Items => _items.AsReadOnly();
    
    public Money Total => _items
        .Select(item => item.SubTotal)
        .Aggregate(Money.Zero, (sum, price) => sum + price);
    
    private ShoppingCart(CartId id, CustomerId customerId, List<CartItem> items)
    {
        Id = id;
        CustomerId = customerId;
        _items = items;
    }
    
    public static ShoppingCart Create(CartId id, CustomerId customerId) =>
        new ShoppingCart(id, customerId, new List<CartItem>());
}

// Pattern 4: Hierarchical composition
public sealed record Organization
{
    public OrganizationId Id { get; }
    public OrganizationName Name { get; }
    public ContactInfo ContactInfo { get; }
    public Option<Organization> ParentOrganization { get; }
    public IReadOnlyList<Department> Departments { get; }
    
    // Complex object built from layers of value objects
}
```

## Why It's a Problem

1. **Scattered validation**: Business rules for concepts duplicated
2. **Weak encapsulation**: Related values not grouped
3. **Hard to reuse**: Can't use "address" concept in multiple entities
4. **Testing complexity**: Must test validation in many places

## Symptoms

- Classes with 10+ primitive properties
- Validation logic duplicated across entities
- Methods taking 5+ string parameters
- Difficulty reusing domain concepts

## Benefits

- **Encapsulation**: Related values bundled with their rules
- **Reusability**: Compose value objects across entities
- **Testability**: Test each value object independently
- **Clarity**: Domain concepts explicit in type system
- **Composability**: Build complex concepts from simple ones

## See Also

- [Primitive Obsession](./primitive-obsession.md) — problem this solves
- [Data Clumps](./data-clump.md) — values that travel together
- [Value Semantics](./value-semantics.md) — immutable value objects
- [Smart Constructors](./smart-constructors.md) — parse, don't validate
- [Domain Invariants](./domain-invariants.md) — enforcing business rules
