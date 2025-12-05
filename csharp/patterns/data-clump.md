# Data Clumps

> Values that always travel together appear as separate parameters.

## Problem

When related values are passed around separately, validation logic becomes "homeless"—it gets duplicated across services, and there's no natural place for cross-field validation rules.

## Example

### ❌ Before

```csharp
public sealed class ShippingService
{
    public void Ship(StreetAddress streetAddress, CityName city, State state, ZipCode zip)
    {
        if (!ZipCodeMatchesAddress(streetAddress, city, state, zip))
            throw new InvalidAddressException();
        // ...
    }

    bool ZipCodeMatchesAddress(StreetAddress streetAddress, CityName city, State state, ZipCode zip)
    {
        // ...
    }
}

// What happens when we add InvoiceService?
sealed class InvoiceService
{
    public void SendBillInSnailMail(StreetAddress streetAddress, CityName city, State state, ZipCode zip)
    {
        // Uh oh... Copy/Paste ZipCodeMatchesAddress? :(
    }
}
```

### ✅ After

```csharp
public sealed record Address
{
    public StreetAddress Street { get; }
    public CityName City { get; }
    public State State { get; }
    public ZipCode Zip { get; }

    public Address(StreetAddress street, CityName city, State state, ZipCode zip)
    {
        if (!ZipMatchesAddress(street, city, state, zip))
            throw new InvalidAddressException();

        Street = street;
        City = city;
        State = state;
        Zip = zip;
    }

    private static bool ZipMatchesAddress(StreetAddress street, CityName city, State state, ZipCode zip)
    {
        // ...
    }
}

public sealed class ShippingService
{
    public void Ship(Address address) { /* ... */ }
}

public sealed class InvoiceService
{
    public void SendBillInSnailMail(Address address) { /* ... */ }
}
```

## Symptoms

- Parameters that always appear together in method signatures
- Long parameter lists with related types
- Validation logic that depends on multiple arguments and/or fields
- Parameter lists that don't match how you talk about the system (do you say "street, city, state, zip" or just "address"?)

## Benefits

- **Shorter parameter lists** reduce cognitive overhead
- **Consistent vocabulary** enables developers to think and communicate clearly
- **Centralized validation** provides a home for cross-field rules
- **Prevents duplication** of validation logic across services

## See Also

- [Primitive Obsession](./primitive-obsession.md)
