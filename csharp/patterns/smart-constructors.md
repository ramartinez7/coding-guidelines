# Smart Constructors (Parse, Don't Validate)

> Validation that returns boolean forces callers to check and remember—use smart constructors that transform untrusted input into trusted types.

## Problem

Traditional validation returns `bool` or throws exceptions, leaving the burden on callers to check results and remember what was validated. Smart constructors "parse" input into a type that carries proof of validity, eliminating the need for downstream checks.

## Example

### ❌ Before

```csharp
public static class Validator
{
    // Validation returns boolean—caller must check
    public static bool IsValidEmail(string email)
    {
        return !string.IsNullOrWhiteSpace(email) &&
               email.Contains("@") &&
               email.Contains(".");
    }
    
    public static bool IsValidAge(int age)
    {
        return age >= 0 && age <= 150;
    }
}

// Caller must remember to validate
public class UserService
{
    public void RegisterUser(string email, int age)
    {
        // Easy to forget validation
        if (!Validator.IsValidEmail(email))
            throw new ArgumentException("Invalid email");
        
        if (!Validator.IsValidAge(age))
            throw new ArgumentException("Invalid age");
        
        // Even after validation, email is still just a string
        // Nothing prevents passing it to methods that expect valid emails
        SendWelcomeEmail(email);
        
        // Downstream methods must trust caller validated
        var user = new User { Email = email, Age = age };
        _repository.Save(user);
    }
    
    // Must trust caller validated email
    private void SendWelcomeEmail(string email) { }
}

// Validation can be bypassed
var user = new User 
{ 
    Email = "not-an-email",  // Invalid!
    Age = -5  // Invalid!
};
```

**Problems:**
- Validation is optional—callers can forget
- Boolean result doesn't carry proof of validity
- Downstream code must trust validation happened
- Easy to bypass validation
- No type safety after validation

### ✅ After

```csharp
/// <summary>
/// Smart constructor: creates EmailAddress only if valid.
/// Once you have an EmailAddress, it's guaranteed valid.
/// </summary>
public sealed record EmailAddress
{
    private const string EmailPattern = @"^[^@\s]+@[^@\s]+\.[^@\s]+$";
    
    public string Value { get; }
    
    // Private constructor—can only create through Parse
    private EmailAddress(string value) => Value = value;
    
    /// <summary>
    /// Parse: transforms string into EmailAddress or returns error.
    /// The type itself is proof of validity.
    /// </summary>
    public static Result<EmailAddress, string> Parse(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return Result<EmailAddress, string>.Failure("Email cannot be empty");
        
        if (!Regex.IsMatch(input, EmailPattern))
            return Result<EmailAddress, string>.Failure(
                $"Invalid email format: {input}");
        
        return Result<EmailAddress, string>.Success(new EmailAddress(input));
    }
    
    public override string ToString() => Value;
}

/// <summary>
/// Smart constructor for age with business rules.
/// </summary>
public readonly record struct Age
{
    public int Value { get; }
    
    private Age(int value) => Value = value;
    
    public static Result<Age, string> Parse(int input)
    {
        if (input < 0)
            return Result<Age, string>.Failure("Age cannot be negative");
        
        if (input > 150)
            return Result<Age, string>.Failure("Age cannot exceed 150");
        
        return Result<Age, string>.Success(new Age(input));
    }
    
    public override string ToString() => Value.ToString();
}

// User entity accepts only validated types
public sealed class User
{
    public UserId Id { get; }
    public EmailAddress Email { get; private set; }
    public Age Age { get; private set; }
    
    private User(UserId id, EmailAddress email, Age age)
    {
        Id = id;
        Email = email;
        Age = age;
    }
    
    public static User Create(EmailAddress email, Age age)
    {
        // No validation needed—types carry proof
        return new User(UserId.New(), email, age);
    }
    
    public void ChangeEmail(EmailAddress newEmail)
    {
        // Type system guarantees validity
        Email = newEmail;
    }
}

public class UserService
{
    public Result<User, string> RegisterUser(string emailInput, int ageInput)
    {
        // Parse inputs into validated types
        var emailResult = EmailAddress.Parse(emailInput);
        if (!emailResult.IsSuccess)
            return Result<User, string>.Failure(emailResult.Error);
        
        var ageResult = Age.Parse(ageInput);
        if (!ageResult.IsSuccess)
            return Result<User, string>.Failure(ageResult.Error);
        
        // Once parsed, types carry proof of validity
        var email = emailResult.Value;
        var age = ageResult.Value;
        
        // No validation needed—types are already valid
        var user = User.Create(email, age);
        SendWelcomeEmail(email);
        
        return Result<User, string>.Success(user);
    }
    
    // Type signature proves email is valid
    private void SendWelcomeEmail(EmailAddress email)
    {
        // No validation needed!
        Console.WriteLine($"Sending to {email}");
    }
}

// ✗ Cannot create invalid user
// var user = new User(UserId.New(), "not-an-email", -5);  // Won't compile!
```

## Parse, Don't Validate Philosophy

| Validate (❌) | Parse (✅) |
|---------------|-----------|
| Returns `bool` | Returns `Result<T, Error>` |
| Callers must check | Type carries proof |
| No compile-time safety | Compile-time guarantees |
| Can be forgotten | Cannot be bypassed |
| Validation repeated | Validate once at boundary |

```csharp
// ❌ Validate: returns boolean
public static bool IsValidPhoneNumber(string phone) { }

// Caller must check and remember
if (IsValidPhoneNumber(phone))
{
    // phone is still a string—no proof it's valid
    SendSms(phone);
}

// ✅ Parse: returns typed value
public static Result<PhoneNumber, string> ParsePhoneNumber(string phone) { }

// Type proves validity
var result = PhoneNumber.Parse(phone);
if (result.IsSuccess)
{
    // result.Value is PhoneNumber—guaranteed valid
    SendSms(result.Value);
}
```

## Smart Constructor Patterns

### 1. Simple Smart Constructor

```csharp
public sealed record Username
{
    private const int MinLength = 3;
    private const int MaxLength = 20;
    private static readonly Regex ValidPattern = new(@"^[a-zA-Z0-9_]+$");
    
    public string Value { get; }
    
    private Username(string value) => Value = value;
    
    public static Result<Username, string> Parse(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return Result<Username, string>.Failure("Username cannot be empty");
        
        if (input.Length < MinLength)
            return Result<Username, string>.Failure(
                $"Username must be at least {MinLength} characters");
        
        if (input.Length > MaxLength)
            return Result<Username, string>.Failure(
                $"Username cannot exceed {MaxLength} characters");
        
        if (!ValidPattern.IsMatch(input))
            return Result<Username, string>.Failure(
                "Username can only contain letters, numbers, and underscores");
        
        return Result<Username, string>.Success(new Username(input));
    }
    
    public override string ToString() => Value;
}
```

### 2. Smart Constructor with Normalization

```csharp
public sealed record PostalCode
{
    private static readonly Regex UsZipPattern = new(@"^\d{5}(-\d{4})?$");
    
    public string Value { get; }
    
    private PostalCode(string value) => Value = value;
    
    /// <summary>
    /// Parse and normalize: "12345-0000" becomes "12345"
    /// </summary>
    public static Result<PostalCode, string> Parse(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return Result<PostalCode, string>.Failure("Postal code cannot be empty");
        
        var trimmed = input.Trim();
        
        if (!UsZipPattern.IsMatch(trimmed))
            return Result<PostalCode, string>.Failure(
                "Invalid US postal code format");
        
        // Normalize: strip unnecessary "-0000" suffix
        var normalized = trimmed.EndsWith("-0000") 
            ? trimmed[..5] 
            : trimmed;
        
        return Result<PostalCode, string>.Success(new PostalCode(normalized));
    }
}

// Usage
var zip1 = PostalCode.Parse("12345");       // "12345"
var zip2 = PostalCode.Parse("12345-0000");  // "12345" (normalized)
var zip3 = PostalCode.Parse("12345-6789");  // "12345-6789"
```

### 3. Smart Constructor with Business Rules

```csharp
public sealed record Percentage
{
    public decimal Value { get; }
    
    private Percentage(decimal value) => Value = value;
    
    public static Result<Percentage, string> Parse(decimal input)
    {
        if (input < 0)
            return Result<Percentage, string>.Failure(
                "Percentage cannot be negative");
        
        if (input > 100)
            return Result<Percentage, string>.Failure(
                "Percentage cannot exceed 100");
        
        return Result<Percentage, string>.Success(new Percentage(input));
    }
    
    public static Percentage FromDecimal(decimal value)
    {
        // For internal use where value is already known to be valid
        return new Percentage(value * 100);
    }
    
    public decimal AsDecimal() => Value / 100;
    public override string ToString() => $"{Value}%";
}

// Usage in business logic
public sealed record Discount
{
    public Percentage Rate { get; }
    
    public static Result<Discount, string> Create(decimal rate)
    {
        var percentageResult = Percentage.Parse(rate);
        if (!percentageResult.IsSuccess)
            return Result<Discount, string>.Failure(percentageResult.Error);
        
        return Result<Discount, string>.Success(
            new Discount { Rate = percentageResult.Value });
    }
    
    public Money ApplyTo(Money amount)
    {
        var discountAmount = amount.Amount * Rate.AsDecimal();
        return new Money(amount.Amount - discountAmount, amount.Currency);
    }
}
```

### 4. Smart Constructor with Multiple Formats

```csharp
public sealed record PhoneNumber
{
    private static readonly Regex UsFormatPattern = new(@"^\d{3}-\d{3}-\d{4}$");
    private static readonly Regex DigitsOnlyPattern = new(@"^\d{10}$");
    
    public string Value { get; }
    
    private PhoneNumber(string value) => Value = value;
    
    public static Result<PhoneNumber, string> Parse(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return Result<PhoneNumber, string>.Failure("Phone number cannot be empty");
        
        var cleaned = input.Trim();
        
        // Accept formatted: "555-123-4567"
        if (UsFormatPattern.IsMatch(cleaned))
            return Result<PhoneNumber, string>.Success(new PhoneNumber(cleaned));
        
        // Accept digits only: "5551234567"
        if (DigitsOnlyPattern.IsMatch(cleaned))
        {
            // Normalize to formatted version
            var formatted = $"{cleaned[..3]}-{cleaned[3..6]}-{cleaned[6..]}";
            return Result<PhoneNumber, string>.Success(new PhoneNumber(formatted));
        }
        
        return Result<PhoneNumber, string>.Failure(
            "Invalid US phone number format");
    }
    
    public string DigitsOnly() => Value.Replace("-", "");
    public override string ToString() => Value;
}
```

### 5. Smart Constructor with Dependent Validation

```csharp
public sealed record CreditCardNumber
{
    public string Value { get; }
    
    private CreditCardNumber(string value) => Value = value;
    
    public static Result<CreditCardNumber, string> Parse(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return Result<CreditCardNumber, string>.Failure(
                "Credit card number cannot be empty");
        
        var digitsOnly = new string(input.Where(char.IsDigit).ToArray());
        
        if (digitsOnly.Length < 13 || digitsOnly.Length > 19)
            return Result<CreditCardNumber, string>.Failure(
                "Credit card number must be 13-19 digits");
        
        // Luhn algorithm validation
        if (!IsValidLuhn(digitsOnly))
            return Result<CreditCardNumber, string>.Failure(
                "Invalid credit card number (checksum failed)");
        
        return Result<CreditCardNumber, string>.Success(
            new CreditCardNumber(digitsOnly));
    }
    
    private static bool IsValidLuhn(string number)
    {
        int sum = 0;
        bool alternate = false;
        
        for (int i = number.Length - 1; i >= 0; i--)
        {
            int digit = number[i] - '0';
            
            if (alternate)
            {
                digit *= 2;
                if (digit > 9)
                    digit -= 9;
            }
            
            sum += digit;
            alternate = !alternate;
        }
        
        return sum % 10 == 0;
    }
    
    public string MaskedValue => $"****-****-****-{Value[^4..]}";
}
```

## Combining Smart Constructors

```csharp
// Multiple smart constructors compose
public sealed class Payment
{
    public PaymentId Id { get; }
    public Money Amount { get; }
    public CreditCardNumber CardNumber { get; }
    public EmailAddress Email { get; }
    
    private Payment(
        PaymentId id,
        Money amount,
        CreditCardNumber cardNumber,
        EmailAddress email)
    {
        Id = id;
        Amount = amount;
        CardNumber = cardNumber;
        Email = email;
    }
    
    public static Result<Payment, IReadOnlyList<string>> Create(
        decimal amount,
        string currency,
        string cardNumber,
        string email)
    {
        var errors = new List<string>();
        
        // Parse all inputs
        var amountResult = Money.Parse(amount, currency);
        if (!amountResult.IsSuccess)
            errors.Add(amountResult.Error);
        
        var cardResult = CreditCardNumber.Parse(cardNumber);
        if (!cardResult.IsSuccess)
            errors.Add(cardResult.Error);
        
        var emailResult = EmailAddress.Parse(email);
        if (!emailResult.IsSuccess)
            errors.Add(emailResult.Error);
        
        // Return all errors at once
        if (errors.Count > 0)
            return Result<Payment, IReadOnlyList<string>>.Failure(errors);
        
        // All inputs valid—create payment
        return Result<Payment, IReadOnlyList<string>>.Success(
            new Payment(
                PaymentId.New(),
                amountResult.Value,
                cardResult.Value,
                emailResult.Value));
    }
}
```

## Benefits

- **Type-driven correctness**: Types carry proof of validity
- **Parse once, use everywhere**: No repeated validation
- **Compile-time safety**: Invalid types don't compile
- **Self-documenting**: Function signatures reveal requirements
- **Fail fast**: Invalid input rejected at boundary
- **Impossible to bypass**: Can only create through smart constructor

## Comparison

```csharp
// ❌ Validate: easy to misuse
public void ProcessOrder(string email, int age)
{
    // Forgot to validate!
    SendEmail(email);  // Runtime error if email invalid
}

// ✅ Parse: impossible to misuse
public void ProcessOrder(EmailAddress email, Age age)
{
    // email and age are guaranteed valid by type system
    SendEmail(email);  // Always safe
}
```

## Why It's a Problem

1. **Validation can be forgotten**: Boolean returns don't enforce checking
2. **No compile-time safety**: Validated data still primitive types
3. **Repeated validation**: Downstream code must re-validate or trust caller
4. **Runtime errors**: Invalid data discovered late
5. **Poor API design**: Function signatures don't reveal validation requirements

## Symptoms

- Methods checking `IsValid*` at the start
- Comments like "caller must validate before calling"
- Validation logic duplicated across methods
- Try-catch blocks for validation errors
- Defensive validation checks everywhere

## See Also

- [Domain Invariants](./domain-invariants.md) — enforcing business rules
- [Honest Functions](./honest-functions.md) — Result types for errors
- [Static Factory Methods](./static-factory-methods.md) — construction patterns
- [Value Semantics](./value-semantics.md) — immutable validated types
- [Input Sanitization](./input-sanitization.md) — trusted types at boundaries
