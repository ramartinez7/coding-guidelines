# Type-Safe Configuration (Compile-Time Configuration Validation)

> Configuration loaded as strings and dictionaries fails at runtime—use strongly-typed configuration with validation to catch errors at startup.

## Problem

Configuration stored in dictionaries or weakly-typed objects allows typos and missing values to fail in production at runtime.

## Example

### ❌ Before

```csharp
// Runtime configuration errors
public class OrderService
{
    private readonly IConfiguration _config;
    
    public async Task ProcessOrder()
    {
        var apiKey = _config["PaymentGateway:ApiKey"];  // Null if missing!
        var timeout = int.Parse(_config["PaymentGateway:TimeoutSeconds"]);  // Exception if invalid!
        var endpoint = _config["PaymentGatewy:Endpoint"];  // Typo! Returns null
    }
}
```

### ✅ After

```csharp
// Type-safe configuration
public sealed record PaymentGatewayConfig
{
    public ApiKey ApiKey { get; init; }
    public TimeSpan Timeout { get; init; }
    public Uri Endpoint { get; init; }
    
    public static Result<PaymentGatewayConfig, string> Create(
        string apiKey,
        int timeoutSeconds,
        string endpoint)
    {
        var apiKeyResult = ApiKey.Create(apiKey);
        if (!apiKeyResult.IsSuccess)
            return Result<PaymentGatewayConfig, string>.Failure(apiKeyResult.Error);
        
        if (timeoutSeconds <= 0)
            return Result<PaymentGatewayConfig, string>.Failure("Timeout must be positive");
        
        if (!Uri.TryCreate(endpoint, UriKind.Absolute, out var uri))
            return Result<PaymentGatewayConfig, string>.Failure("Invalid endpoint URL");
        
        return Result<PaymentGatewayConfig, string>.Success(new PaymentGatewayConfig
        {
            ApiKey = apiKeyResult.Value,
            Timeout = TimeSpan.FromSeconds(timeoutSeconds),
            Endpoint = uri
        });
    }
}

// Validate at startup
public static class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);
        
        var configResult = PaymentGatewayConfig.Create(
            builder.Configuration["PaymentGateway:ApiKey"],
            int.Parse(builder.Configuration["PaymentGateway:TimeoutSeconds"]),
            builder.Configuration["PaymentGateway:Endpoint"]);
        
        if (!configResult.IsSuccess)
            throw new InvalidOperationException($"Invalid configuration: {configResult.Error}");
        
        builder.Services.AddSingleton(configResult.Value);
        
        var app = builder.Build();
        app.Run();
    }
}
```

## See Also

- [Validated Configuration](./validated-configuration.md)
- [Smart Constructors](./smart-constructors.md)
