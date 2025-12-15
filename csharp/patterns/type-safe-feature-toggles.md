# Type-Safe Feature Toggles (Compile-Time Feature Flags)

> Boolean feature flags checked at runtime allow dead code and untested paths—use types to make features explicit and eliminate runtime checks.

## Problem

Feature flags implemented as boolean configuration values require runtime checks scattered throughout the codebase. Features can be partially implemented, flags can be forgotten, and the compiler can't verify all code paths are implemented.

## Example

### ❌ Before

```csharp
public class PaymentService
{
    private readonly IConfiguration _config;
    
    public async Task ProcessPayment(Payment payment)
    {
        if (_config.GetValue<bool>("Features:UseNewPaymentGateway"))
        {
            await ProcessWithNewGateway(payment);
        }
        else
        {
            await ProcessWithOldGateway(payment);
        }
    }
}
```

### ✅ After

```csharp
public interface IPaymentGateway
{
    Task ProcessAsync(Payment payment);
}

public sealed class LegacyPaymentGateway : IPaymentGateway
{
    public Task ProcessAsync(Payment payment) { /* old implementation */ return Task.CompletedTask; }
}

public sealed class ModernPaymentGateway : IPaymentGateway
{
    public Task ProcessAsync(Payment payment) { /* new implementation */ return Task.CompletedTask; }
}

// Registration based on configuration
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        var useNewGateway = Configuration.GetValue<bool>("Features:UseNewPaymentGateway");
        
        if (useNewGateway)
            services.AddScoped<IPaymentGateway, ModernPaymentGateway>();
        else
            services.AddScoped<IPaymentGateway, LegacyPaymentGateway>();
    }
}

public class PaymentService
{
    private readonly IPaymentGateway _gateway;
    
    public PaymentService(IPaymentGateway gateway)
    {
        _gateway = gateway;
    }
    
    public Task ProcessPayment(Payment payment)
    {
        return _gateway.ProcessAsync(payment);  // No runtime checks
    }
}
```

## See Also

- [Polymorphic Feature Flags](./polymorphic-feature-flags.md)
- [Dependency Injection](./dependency-injection.md)
