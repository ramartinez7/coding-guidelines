# Infrastructure Service Adapters

> Implement infrastructure concerns (email, file storage, external APIs) as adapters that implement domain or application interfaces—keeping infrastructure swappable and domain pure.

## Problem

When domain or application logic directly depends on specific infrastructure implementations (SendGrid, AWS S3, Stripe API), it becomes tightly coupled to those technologies and hard to test or swap.

## Core Concept

Define interfaces in domain/application layers, implement them with adapters in infrastructure layer:

```
Domain Layer: interface IEmailService { }
Application Layer: uses IEmailService
Infrastructure Layer: class SendGridEmailAdapter : IEmailService { }
                     class SmtpEmailAdapter : IEmailService { }
                     class ConsoleEmailAdapter : IEmailService { } (for testing)
```

## Example

### ❌ Before (Direct Infrastructure Dependency)

```csharp
// Application layer directly depends on SendGrid
public class RegisterUserUseCase
{
    private readonly ISendGridClient _sendGridClient;  // ❌ Specific technology
    
    public async Task ExecuteAsync(RegisterUserCommand command)
    {
        var user = User.Register(command.Email, command.Password);
        
        // ❌ SendGrid-specific code in application layer
        var message = new SendGridMessage
        {
            From = new EmailAddress("noreply@myapp.com"),
            Subject = "Welcome!",
            PlainTextContent = $"Welcome {user.Name}!"
        };
        message.AddTo(new EmailAddress(user.Email));
        
        await _sendGridClient.SendEmailAsync(message);
    }
}
```

### ✅ After (Infrastructure Adapters)

```csharp
// Application/Interfaces/IEmailService.cs (Application layer)
namespace MyApp.Application.Interfaces
{
    /// <summary>
    /// Abstract email service - no knowledge of implementation.
    /// </summary>
    public interface IEmailService
    {
        Task SendWelcomeEmailAsync(Email recipient, string userName);
        Task SendPasswordResetEmailAsync(Email recipient, string resetToken);
        Task SendOrderConfirmationAsync(Email recipient, OrderId orderId);
    }
}

// Application/UseCases/RegisterUser/RegisterUserUseCase.cs
public sealed class RegisterUserUseCase
{
    private readonly IUserRepository _userRepository;
    private readonly IEmailService _emailService;  // ✅ Abstract interface
    
    public async Task<Result<UserId, RegisterUserError>> ExecuteAsync(
        RegisterUserCommand command)
    {
        var user = User.Register(command.Email, command.Password);
        
        await _userRepository.SaveAsync(user);
        
        // ✅ High-level email abstraction
        await _emailService.SendWelcomeEmailAsync(user.Email, user.Name);
        
        return Result<UserId, RegisterUserError>.Success(user.Id);
    }
}

// Infrastructure/Email/SendGridEmailAdapter.cs
namespace MyApp.Infrastructure.Email
{
    /// <summary>
    /// SendGrid implementation of IEmailService.
    /// </summary>
    public sealed class SendGridEmailAdapter : IEmailService
    {
        private readonly ISendGridClient _client;
        private readonly EmailConfiguration _config;
        
        public async Task SendWelcomeEmailAsync(Email recipient, string userName)
        {
            var message = new SendGridMessage
            {
                From = new EmailAddress(_config.FromAddress, _config.FromName),
                Subject = "Welcome to MyApp!",
                PlainTextContent = $"Welcome {userName}!",
                HtmlContent = $"<h1>Welcome {userName}!</h1>"
            };
            message.AddTo(new EmailAddress(recipient.Value));
            
            await _client.SendEmailAsync(message);
        }
        
        public async Task SendPasswordResetEmailAsync(Email recipient, string resetToken)
        {
            // SendGrid-specific implementation
        }
        
        public async Task SendOrderConfirmationAsync(Email recipient, OrderId orderId)
        {
            // SendGrid-specific implementation
        }
    }
}

// Infrastructure/Email/SmtpEmailAdapter.cs
namespace MyApp.Infrastructure.Email
{
    /// <summary>
    /// SMTP implementation - can swap with SendGrid without changing app logic.
    /// </summary>
    public sealed class SmtpEmailAdapter : IEmailService
    {
        private readonly SmtpClient _smtpClient;
        
        public async Task SendWelcomeEmailAsync(Email recipient, string userName)
        {
            var message = new MailMessage
            {
                From = new MailAddress("noreply@myapp.com"),
                Subject = "Welcome to MyApp!",
                Body = $"Welcome {userName}!",
                To = { new MailAddress(recipient.Value) }
            };
            
            await _smtpClient.SendMailAsync(message);
        }
        
        // Other methods...
    }
}

// Infrastructure/Email/ConsoleEmailAdapter.cs (for development/testing)
namespace MyApp.Infrastructure.Email
{
    /// <summary>
    /// Fake email service that logs to console - perfect for testing.
    /// </summary>
    public sealed class ConsoleEmailAdapter : IEmailService
    {
        private readonly ILogger<ConsoleEmailAdapter> _logger;
        
        public Task SendWelcomeEmailAsync(Email recipient, string userName)
        {
            _logger.LogInformation(
                "EMAIL: Welcome email to {Recipient} (Name: {UserName})",
                recipient.Value,
                userName);
            
            return Task.CompletedTask;
        }
        
        // Other methods...
    }
}

// DI Configuration
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        var emailProvider = configuration.GetValue<string>("EmailProvider");
        
        switch (emailProvider)
        {
            case "SendGrid":
                services.AddScoped<IEmailService, SendGridEmailAdapter>();
                break;
            case "Smtp":
                services.AddScoped<IEmailService, SmtpEmailAdapter>();
                break;
            case "Console":
                services.AddScoped<IEmailService, ConsoleEmailAdapter>();
                break;
            default:
                throw new InvalidOperationException($"Unknown email provider: {emailProvider}");
        }
        
        return services;
    }
}
```

## More Adapter Examples

### File Storage Adapter

```csharp
// Application interface
public interface IFileStorageService
{
    Task<string> UploadAsync(string fileName, Stream content);
    Task<Stream> DownloadAsync(string fileId);
    Task DeleteAsync(string fileId);
}

// AWS S3 Adapter
public sealed class S3FileStorageAdapter : IFileStorageService
{
    private readonly IAmazonS3 _s3Client;
    private readonly string _bucketName;
    
    public async Task<string> UploadAsync(string fileName, Stream content)
    {
        var key = $"{Guid.NewGuid()}/{fileName}";
        
        await _s3Client.PutObjectAsync(new PutObjectRequest
        {
            BucketName = _bucketName,
            Key = key,
            InputStream = content
        });
        
        return key;
    }
}

// Azure Blob Storage Adapter
public sealed class AzureBlobStorageAdapter : IFileStorageService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly string _containerName;
    
    public async Task<string> UploadAsync(string fileName, Stream content)
    {
        var container = _blobServiceClient.GetBlobContainerClient(_containerName);
        var blobClient = container.GetBlobClient(fileName);
        
        await blobClient.UploadAsync(content);
        
        return blobClient.Uri.ToString();
    }
}
```

### Payment Gateway Adapter

```csharp
// Application interface
public interface IPaymentGateway
{
    Task<Result<PaymentId, PaymentError>> ProcessPaymentAsync(
        Money amount,
        PaymentMethod paymentMethod);
}

// Stripe Adapter
public sealed class StripePaymentAdapter : IPaymentGateway
{
    private readonly IStripeClient _stripeClient;
    
    public async Task<Result<PaymentId, PaymentError>> ProcessPaymentAsync(
        Money amount,
        PaymentMethod paymentMethod)
    {
        var chargeOptions = new ChargeCreateOptions
        {
            Amount = (long)(amount.Amount * 100),
            Currency = amount.Currency.Code.ToLower(),
            Source = paymentMethod.Token
        };
        
        var charge = await _stripeClient.Charges.CreateAsync(chargeOptions);
        
        return Result<PaymentId, PaymentError>.Success(
            new PaymentId(charge.Id));
    }
}
```

## Benefits

1. **Swappable**: Change infrastructure without touching business logic
2. **Testable**: Use fake adapters for testing
3. **Development**: Use console/in-memory adapters during development
4. **Technology Independence**: Domain never knows about SendGrid, S3, Stripe

## See Also

- [Port-Adapter Separation](./port-adapter-separation.md) — hexagonal architecture
- [Clean Architecture](./clean-architecture.md) — layered dependencies
- [Anti-Corruption Layer](./anti-corruption-layer.md) — protecting domain
