# Discriminated Unions (Modeling Mutually Exclusive Alternatives)

> Using multiple nullable fields or flags to represent alternatives—use discriminated unions (sum types) to model "one of" relationships where only one variant can exist at a time.

## Problem

When an entity can be in one of several mutually exclusive states, traditional approaches use multiple nullable fields, boolean flags, or state enums. This allows invalid states where multiple alternatives are set, or none are set, creating runtime bugs.

## Example

### ❌ Before

```csharp
// Multiple nullable fields allow invalid combinations
public class ContactMethod
{
    public string? Email { get; set; }
    public string? Phone { get; set; }
    public string? PostalAddress { get; set; }
    
    // Which one is valid? All three? None? Type doesn't say.
}

// Problems:
var contact = new ContactMethod
{
    Email = "user@example.com",
    Phone = "+1234567890",
    PostalAddress = null
};
// Is this valid? Should we use email or phone?

// Or with flags:
public class Payment
{
    public bool IsCreditCard { get; set; }
    public bool IsBankTransfer { get; set; }
    public bool IsCash { get; set; }
    
    public string? CardNumber { get; set; }
    public string? AccountNumber { get; set; }
    public decimal? CashAmount { get; set; }
}

// What if all flags are true? Or none?
var payment = new Payment
{
    IsCreditCard = true,
    IsBankTransfer = true,  // Impossible! Can't be both
    CardNumber = "4111",
    AccountNumber = "9876"  // Which one is valid?
};
```

**Problems:**
- Can set multiple alternatives simultaneously
- Can set no alternatives (all null)
- Fields valid for one variant can be set in another
- Runtime validation needed everywhere
- Type doesn't express "one and only one"

### ✅ After (Discriminated Union)

```csharp
/// <summary>
/// Contact method - exactly one variant at a time.
/// Sealed hierarchy makes illegal states unrepresentable.
/// </summary>
public abstract record ContactMethod
{
    // Private constructor ensures all variants defined here
    private ContactMethod() { }
    
    public sealed record Email(string Address) : ContactMethod
    {
        public static Result<Email, ValidationError> Create(string address)
        {
            if (!address.Contains('@'))
                return Result<Email, ValidationError>.Failure(
                    new ValidationError("Invalid email format"));
            
            return Result<Email, ValidationError>.Success(new Email(address));
        }
    }
    
    public sealed record Phone(string Number, string? Extension) : ContactMethod
    {
        public static Result<Phone, ValidationError> Create(string number)
        {
            // Phone validation
            if (number.Length < 10)
                return Result<Phone, ValidationError>.Failure(
                    new ValidationError("Phone number too short"));
            
            return Result<Phone, ValidationError>.Success(new Phone(number, null));
        }
    }
    
    public sealed record PostalAddress(
        string Street,
        string City,
        string State,
        string ZipCode) : ContactMethod;
}

// Usage: Impossible to have multiple contact types
var emailContact = ContactMethod.Email.Create("user@example.com");
var phoneContact = ContactMethod.Phone.Create("+1234567890");

// Pattern matching handles all cases
string FormatContact(ContactMethod contact) => contact switch
{
    ContactMethod.Email email => $"Email: {email.Address}",
    ContactMethod.Phone phone => phone.Extension is not null
        ? $"Phone: {phone.Number} ext. {phone.Extension}"
        : $"Phone: {phone.Number}",
    ContactMethod.PostalAddress addr =>
        $"Mail: {addr.Street}, {addr.City}, {addr.State} {addr.ZipCode}"
};
```

## Payment Methods as Discriminated Union

```csharp
/// <summary>
/// Payment - exactly one method with its specific data.
/// </summary>
public abstract record Payment
{
    private Payment() { }
    
    public sealed record CreditCard(
        string CardNumber,
        string Cvv,
        DateTime Expiry,
        string CardholderName) : Payment
    {
        // Only credit cards have these fields
        public string Last4Digits => CardNumber[^4..];
        
        public bool IsExpired => DateTime.UtcNow > Expiry;
    }
    
    public sealed record BankTransfer(
        string AccountNumber,
        string RoutingNumber,
        string BankName) : Payment;
    
    public sealed record Cash(decimal Amount) : Payment;
    
    public sealed record DigitalWallet(
        string WalletProvider,
        string WalletId) : Payment;
}

public class PaymentProcessor
{
    public Result<PaymentConfirmation, PaymentError> ProcessPayment(
        Payment payment,
        decimal amount)
    {
        // Exhaustive matching ensures all payment types handled
        return payment switch
        {
            Payment.CreditCard cc => ProcessCreditCard(cc, amount),
            Payment.BankTransfer bt => ProcessBankTransfer(bt, amount),
            Payment.Cash cash => ProcessCash(cash, amount),
            Payment.DigitalWallet wallet => ProcessDigitalWallet(wallet, amount)
        };
    }
    
    Result<PaymentConfirmation, PaymentError> ProcessCreditCard(
        Payment.CreditCard card,
        decimal amount)
    {
        // Card-specific validation
        if (card.IsExpired)
            return Result<PaymentConfirmation, PaymentError>.Failure(
                new PaymentError.CardExpired(card.Last4Digits));
        
        // Process with payment gateway...
        return Result<PaymentConfirmation, PaymentError>.Success(
            new PaymentConfirmation("CC-12345", DateTime.UtcNow));
    }
    
    Result<PaymentConfirmation, PaymentError> ProcessBankTransfer(
        Payment.BankTransfer transfer,
        decimal amount)
    {
        // Bank-specific logic...
        return Result<PaymentConfirmation, PaymentError>.Success(
            new PaymentConfirmation("BT-67890", DateTime.UtcNow));
    }
    
    Result<PaymentConfirmation, PaymentError> ProcessCash(
        Payment.Cash cash,
        decimal amount)
    {
        // Verify amount matches
        if (cash.Amount < amount)
            return Result<PaymentConfirmation, PaymentError>.Failure(
                new PaymentError.InsufficientFunds(cash.Amount, amount));
        
        return Result<PaymentConfirmation, PaymentError>.Success(
            new PaymentConfirmation("CASH-11111", DateTime.UtcNow));
    }
    
    Result<PaymentConfirmation, PaymentError> ProcessDigitalWallet(
        Payment.DigitalWallet wallet,
        decimal amount)
    {
        // Wallet-specific API call...
        return Result<PaymentConfirmation, PaymentError>.Success(
            new PaymentConfirmation($"{wallet.WalletProvider}-22222", DateTime.UtcNow));
    }
}
```

## Search Criteria as Discriminated Union

```csharp
/// <summary>
/// Product search - exactly one search strategy.
/// </summary>
public abstract record ProductSearch
{
    private ProductSearch() { }
    
    public sealed record ById(ProductId Id) : ProductSearch;
    
    public sealed record BySku(string Sku) : ProductSearch;
    
    public sealed record ByName(string Name, bool ExactMatch) : ProductSearch;
    
    public sealed record ByCategory(
        CategoryId CategoryId,
        bool IncludeSubcategories) : ProductSearch;
    
    public sealed record ByPriceRange(
        decimal MinPrice,
        decimal MaxPrice) : ProductSearch;
    
    public sealed record AdvancedSearch(
        string? Name,
        CategoryId? CategoryId,
        decimal? MinPrice,
        decimal? MaxPrice,
        bool? InStock) : ProductSearch;
}

public class ProductRepository
{
    public async Task<IReadOnlyList<Product>> Search(ProductSearch search)
    {
        // Each search type has its own optimized query
        return search switch
        {
            ProductSearch.ById byId =>
                await SearchById(byId.Id),
            
            ProductSearch.BySku bySku =>
                await SearchBySku(bySku.Sku),
            
            ProductSearch.ByName byName =>
                await SearchByName(byName.Name, byName.ExactMatch),
            
            ProductSearch.ByCategory byCat =>
                await SearchByCategory(byCat.CategoryId, byCat.IncludeSubcategories),
            
            ProductSearch.ByPriceRange byPrice =>
                await SearchByPriceRange(byPrice.MinPrice, byPrice.MaxPrice),
            
            ProductSearch.AdvancedSearch advanced =>
                await SearchAdvanced(advanced)
        };
    }
    
    async Task<IReadOnlyList<Product>> SearchById(ProductId id)
    {
        // Most efficient: direct ID lookup
        var product = await _dbContext.Products.FindAsync(id);
        return product is not null ? new[] { product } : Array.Empty<Product>();
    }
    
    // ... other search implementations
}
```

## Authentication Result as Discriminated Union

```csharp
/// <summary>
/// Authentication attempt result - exactly one outcome.
/// </summary>
public abstract record AuthenticationResult
{
    private AuthenticationResult() { }
    
    public sealed record Success(
        UserId UserId,
        string Username,
        IReadOnlyList<string> Roles,
        DateTime AuthenticatedAt) : AuthenticationResult;
    
    public sealed record InvalidCredentials(
        string Username,
        int FailedAttemptCount) : AuthenticationResult;
    
    public sealed record AccountLocked(
        string Username,
        DateTime LockedUntil) : AuthenticationResult;
    
    public sealed record AccountDisabled(
        string Username,
        string Reason) : AuthenticationResult;
    
    public sealed record RequiresTwoFactor(
        string Username,
        string TwoFactorMethod) : AuthenticationResult;
    
    public sealed record RequiresPasswordChange(
        string Username,
        string Reason) : AuthenticationResult;
}

public class AuthenticationService
{
    public async Task<AuthenticationResult> Authenticate(
        string username,
        string password)
    {
        var user = await _userRepository.GetByUsername(username);
        
        if (user is null)
            return new AuthenticationResult.InvalidCredentials(username, 0);
        
        if (user.IsLocked)
            return new AuthenticationResult.AccountLocked(
                username,
                user.LockedUntil);
        
        if (!user.IsEnabled)
            return new AuthenticationResult.AccountDisabled(
                username,
                "Account has been disabled by administrator");
        
        if (!VerifyPassword(user, password))
        {
            await IncrementFailedAttempts(user);
            return new AuthenticationResult.InvalidCredentials(
                username,
                user.FailedLoginAttempts + 1);
        }
        
        if (user.RequiresTwoFactor)
            return new AuthenticationResult.RequiresTwoFactor(
                username,
                user.TwoFactorMethod);
        
        if (user.MustChangePassword)
            return new AuthenticationResult.RequiresPasswordChange(
                username,
                "Password expired");
        
        return new AuthenticationResult.Success(
            user.Id,
            user.Username,
            user.Roles,
            DateTime.UtcNow);
    }
}

public class AuthController
{
    public IActionResult Login(LoginRequest request)
    {
        var result = _authService.Authenticate(request.Username, request.Password);
        
        // Exhaustive matching converts to HTTP responses
        return result switch
        {
            AuthenticationResult.Success success =>
                Ok(new
                {
                    Token = GenerateToken(success.UserId),
                    Username = success.Username,
                    Roles = success.Roles
                }),
            
            AuthenticationResult.InvalidCredentials invalid =>
                Unauthorized(new
                {
                    Message = "Invalid credentials",
                    FailedAttempts = invalid.FailedAttemptCount
                }),
            
            AuthenticationResult.AccountLocked locked =>
                StatusCode(423, new
                {
                    Message = $"Account locked until {locked.LockedUntil}"
                }),
            
            AuthenticationResult.AccountDisabled disabled =>
                Forbidden(new { Message = disabled.Reason }),
            
            AuthenticationResult.RequiresTwoFactor twoFactor =>
                StatusCode(428, new
                {
                    Message = "Two-factor authentication required",
                    Method = twoFactor.TwoFactorMethod
                }),
            
            AuthenticationResult.RequiresPasswordChange pwChange =>
                StatusCode(428, new
                {
                    Message = pwChange.Reason,
                    RequiresPasswordChange = true
                })
        };
    }
}
```

## File Upload Result as Discriminated Union

```csharp
/// <summary>
/// File upload result - exactly one outcome.
/// </summary>
public abstract record FileUploadResult
{
    private FileUploadResult() { }
    
    public sealed record Success(
        FileId FileId,
        string FileName,
        long FileSizeBytes,
        string ContentType,
        string StorageUrl) : FileUploadResult;
    
    public sealed record FileTooLarge(
        long FileSizeBytes,
        long MaxSizeBytes) : FileUploadResult;
    
    public sealed record InvalidFileType(
        string ContentType,
        IReadOnlyList<string> AllowedTypes) : FileUploadResult;
    
    public sealed record VirusScanFailed(
        string FileName,
        string ThreatName) : FileUploadResult;
    
    public sealed record StorageError(
        string FileName,
        string ErrorMessage) : FileUploadResult;
}

public class FileUploadService
{
    const long MaxFileSizeBytes = 10 * 1024 * 1024; // 10 MB
    readonly string[] AllowedTypes = { "image/jpeg", "image/png", "application/pdf" };
    
    public async Task<FileUploadResult> Upload(Stream fileStream, string fileName, string contentType)
    {
        // Check file size
        if (fileStream.Length > MaxFileSizeBytes)
            return new FileUploadResult.FileTooLarge(
                fileStream.Length,
                MaxFileSizeBytes);
        
        // Check file type
        if (!AllowedTypes.Contains(contentType))
            return new FileUploadResult.InvalidFileType(
                contentType,
                AllowedTypes);
        
        // Scan for viruses
        var scanResult = await _virusScanner.Scan(fileStream);
        if (!scanResult.IsClean)
            return new FileUploadResult.VirusScanFailed(
                fileName,
                scanResult.ThreatName);
        
        // Upload to storage
        try
        {
            var fileId = FileId.NewId();
            var storageUrl = await _storageService.Upload(fileId, fileStream);
            
            return new FileUploadResult.Success(
                fileId,
                fileName,
                fileStream.Length,
                contentType,
                storageUrl);
        }
        catch (Exception ex)
        {
            return new FileUploadResult.StorageError(fileName, ex.Message);
        }
    }
}
```

## Discriminated Union with Shared Data

```csharp
/// <summary>
/// Notification - all variants share common metadata,
/// each has type-specific payload.
/// </summary>
public abstract record Notification
{
    public NotificationId Id { get; }
    public UserId UserId { get; }
    public DateTime CreatedAt { get; }
    public bool IsRead { get; }
    
    private Notification(NotificationId id, UserId userId, DateTime createdAt, bool isRead)
    {
        Id = id;
        UserId = userId;
        CreatedAt = createdAt;
        IsRead = isRead;
    }
    
    public sealed record OrderShipped(
        NotificationId Id,
        UserId UserId,
        DateTime CreatedAt,
        bool IsRead,
        OrderId OrderId,
        TrackingNumber TrackingNumber) : Notification(Id, UserId, CreatedAt, IsRead);
    
    public sealed record PaymentReceived(
        NotificationId Id,
        UserId UserId,
        DateTime CreatedAt,
        bool IsRead,
        PaymentId PaymentId,
        decimal Amount) : Notification(Id, UserId, CreatedAt, IsRead);
    
    public sealed record MessageReceived(
        NotificationId Id,
        UserId UserId,
        DateTime CreatedAt,
        bool IsRead,
        UserId FromUserId,
        string MessagePreview) : Notification(Id, UserId, CreatedAt, IsRead);
}

public class NotificationRenderer
{
    public string RenderNotification(Notification notification)
    {
        // Common rendering
        var icon = notification.IsRead ? "✓" : "•";
        var time = FormatTimeAgo(notification.CreatedAt);
        
        // Type-specific message
        var message = notification switch
        {
            Notification.OrderShipped shipped =>
                $"Order {shipped.OrderId} has shipped! Tracking: {shipped.TrackingNumber}",
            
            Notification.PaymentReceived payment =>
                $"Payment of ${payment.Amount:N2} received",
            
            Notification.MessageReceived msg =>
                $"New message from {msg.FromUserId}: {msg.MessagePreview}"
        };
        
        return $"{icon} {message} ({time})";
    }
}
```

## Why It's a Problem

1. **Invalid states**: Multiple alternatives can be set simultaneously
2. **Null handling**: Need to check which field is non-null
3. **Runtime validation**: Must verify exactly one alternative is set
4. **Unclear intent**: Type doesn't express "one of" relationship
5. **Maintenance burden**: Adding variants requires updating validation

## Symptoms

- Multiple nullable properties that are mutually exclusive
- Boolean flags to indicate which variant is active
- State enum with associated nullable fields
- Validation code checking "exactly one field is set"
- Comments like "Only one of X, Y, or Z should be set"

## Benefits

- **Impossible to represent invalid states**: Exactly one variant by construction
- **No null checks**: Each variant has its required data
- **Exhaustive matching**: Compiler ensures all cases handled
- **Self-documenting**: Type clearly shows mutually exclusive alternatives
- **Type-safe**: Cannot access wrong variant's data

## Trade-offs

- **More types**: Each alternative needs a sealed record
- **Pattern matching required**: Must use switch expressions to access data
- **Cannot extend**: Sealed hierarchy prevents external variants
- **Verbosity**: More code than simple class with nullable fields

## See Also

- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md) — replacing enums with types
- [Exhaustive Pattern Matching](./exhaustive-pattern-matching.md) — compiler-enforced completeness
- [Typed Errors](./typed-errors.md) — error cases as discriminated unions
- [Ghost States](./ghost-states.md) — invalid states through partial data
