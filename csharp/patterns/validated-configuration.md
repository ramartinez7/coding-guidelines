# Validated Configuration (The Crash-Early Config)

> Configuration as a "dictionary of strings" crashes hours later when accessed—use strongly-typed options with startup validation to fail fast.

## Problem

.NET configuration (`IConfiguration`) is essentially a `Dictionary<string, string>`. Missing or malformed values (e.g., a URL without a scheme, an invalid email) don't cause errors at startup—the application boots successfully but crashes hours later when the code finally accesses that configuration value.

## Example

### ❌ Before

```csharp
public class EmailService
{
    private readonly IConfiguration _config;

    public EmailService(IConfiguration config)
    {
        _config = config;
    }

    public async Task SendAsync(string to, string subject, string body)
    {
        // 💥 Crashes here—hours after startup—if SmtpHost is missing
        var host = _config["Email:SmtpHost"];
        var port = int.Parse(_config["Email:SmtpPort"]!);  // 💥 FormatException if malformed
        
        using var client = new SmtpClient(host, port);
        // ...
    }
}

// Startup: no validation, no feedback
builder.Services.AddScoped<EmailService>();

// appsettings.json (typo in key name)
{
    "Email": {
        "SmptHost": "smtp.example.com",  // Typo: "Smpt" instead of "Smtp"
        "SmtpPort": "not-a-number"
    }
}
```

**Problems:**
- Application starts successfully despite broken configuration
- Crash occurs when `SendAsync` is first called—possibly hours later
- No indication of *which* configuration is wrong
- Raw strings scattered throughout the codebase

### ✅ After

```csharp
// Strongly-typed options class with validation
public sealed class EmailOptions
{
    public const string SectionName = "Email";

    [Required]
    public required string SmtpHost { get; init; }

    [Required, Range(1, 65535)]
    public required int SmtpPort { get; init; }

    [Required, EmailAddress]
    public required string FromAddress { get; init; }

    [Required, Url]
    public string? UnsubscribeUrl { get; init; }
}

// Service receives validated, typed configuration
public class EmailService
{
    private readonly EmailOptions _options;

    public EmailService(IOptions<EmailOptions> options)
    {
        _options = options.Value;
    }

    public async Task SendAsync(string to, string subject, string body)
    {
        // No parsing, no null checks—options are guaranteed valid
        using var client = new SmtpClient(_options.SmtpHost, _options.SmtpPort);
        // ...
    }
}

// Startup: validate on boot
builder.Services
    .AddOptions<EmailOptions>()
    .BindConfiguration(EmailOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();  // 💥 Fails immediately if invalid
```

**Result:** Application refuses to start if configuration is invalid:

```
Unhandled exception: Microsoft.Extensions.Options.OptionsValidationException:
  DataAnnotation validation failed for 'EmailOptions':
    The SmtpHost field is required.
    The SmtpPort field must be between 1 and 65535.
```

## Why It's a Problem

1. **Late failure**: Broken configuration isn't discovered until runtime—possibly in production, hours after deployment.

2. **Silent corruption**: Typos in key names silently return `null` instead of failing.

3. **Primitive obsession**: Loose `string` and `int` values scatter throughout the codebase without structure.

4. **No discoverability**: Developers must read code to find required configuration keys.

5. **Testing gaps**: Integration tests might pass because they don't exercise the specific code path that reads broken config.

## Symptoms

- `NullReferenceException` or `FormatException` deep in application code
- Configuration-related bugs that only appear in specific environments
- Developers asking "what config keys does this service need?"
- Copy-paste configuration between environments with subtle typos
- Comments like "make sure Email:SmtpHost is set in production"

## Validation Approaches

### Data Annotations (Built-in)

```csharp
public sealed class DatabaseOptions
{
    [Required]
    public required string ConnectionString { get; init; }

    [Range(1, 100)]
    public int MaxPoolSize { get; init; } = 10;

    [RegularExpression(@"^\d+$", ErrorMessage = "Timeout must be numeric")]
    public string? TimeoutMs { get; init; }
}

builder.Services
    .AddOptions<DatabaseOptions>()
    .BindConfiguration("Database")
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

### FluentValidation (Complex Rules)

```csharp
public sealed class ApiOptions
{
    public required Uri BaseUrl { get; init; }
    public required string ApiKey { get; init; }
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
}

public sealed class ApiOptionsValidator : AbstractValidator<ApiOptions>
{
    public ApiOptionsValidator()
    {
        RuleFor(x => x.BaseUrl)
            .NotNull()
            .Must(u => u.Scheme == "https")
            .WithMessage("BaseUrl must use HTTPS");

        RuleFor(x => x.ApiKey)
            .NotEmpty()
            .MinimumLength(32)
            .WithMessage("ApiKey must be at least 32 characters");

        RuleFor(x => x.Timeout)
            .InclusiveBetween(TimeSpan.FromSeconds(1), TimeSpan.FromMinutes(5));
    }
}

// Registration with FluentValidation
builder.Services
    .AddOptions<ApiOptions>()
    .BindConfiguration("ExternalApi")
    .ValidateFluentValidation()  // Extension method
    .ValidateOnStart();
```

### IValidateOptions (Custom Logic)

```csharp
public sealed class FeatureFlagsOptions
{
    public bool EnableNewCheckout { get; init; }
    public bool EnableLegacyCheckout { get; init; }
}

public sealed class FeatureFlagsValidator : IValidateOptions<FeatureFlagsOptions>
{
    public ValidateOptionsResult Validate(string? name, FeatureFlagsOptions options)
    {
        if (options.EnableNewCheckout && options.EnableLegacyCheckout)
        {
            return ValidateOptionsResult.Fail(
                "Cannot enable both NewCheckout and LegacyCheckout simultaneously.");
        }

        if (!options.EnableNewCheckout && !options.EnableLegacyCheckout)
        {
            return ValidateOptionsResult.Fail(
                "At least one checkout system must be enabled.");
        }

        return ValidateOptionsResult.Success;
    }
}

builder.Services.AddSingleton<IValidateOptions<FeatureFlagsOptions>, FeatureFlagsValidator>();
```

## Combining with Domain Types

For maximum safety, parse configuration into domain types:

```csharp
public sealed class EmailOptions
{
    public required string SmtpHost { get; init; }
    public required int SmtpPort { get; init; }
    public required string FromAddress { get; init; }
}

// Domain type that guarantees validity
public sealed record EmailConfiguration(
    HostName SmtpHost,
    Port SmtpPort,
    EmailAddress FromAddress)
{
    public static EmailConfiguration FromOptions(EmailOptions options)
    {
        return new EmailConfiguration(
            HostName.Parse(options.SmtpHost),
            Port.Create(options.SmtpPort),
            EmailAddress.Parse(options.FromAddress));
    }
}

// Register the parsed configuration
builder.Services.AddSingleton(sp =>
{
    var options = sp.GetRequiredService<IOptions<EmailOptions>>().Value;
    return EmailConfiguration.FromOptions(options);
});
```

## Benefits

- **Fail fast**: Invalid configuration crashes at startup, not at 3 AM
- **Discoverability**: Options classes document required configuration
- **Type safety**: No string parsing scattered throughout code
- **Testability**: Options classes can be instantiated directly in tests
- **IDE support**: IntelliSense, refactoring, find-references all work

## See Also

- [Primitive Obsession](./primitive-obsession.md)
- [Static Factory Methods](./static-factory-methods.md)
- [Honest Functions](./honest-functions.md)
