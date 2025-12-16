# Composition Root (DI Container Configuration)

> Configure dependency injection in a single location (composition root) at the application's entry point—wiring together interfaces and implementations while keeping the application core ignorant of DI frameworks.

## Problem

When dependency registration is scattered across multiple projects or the application core knows about DI frameworks, the architecture becomes polluted and hard to maintain.

## Core Concept

**Composition Root** is the single place where:
- All dependencies are wired together
- Interfaces mapped to implementations
- Object graphs constructed
- Located at application entry point (Program.cs, Startup.cs)

Only the composition root knows about the DI container. The application core remains DI-framework agnostic.

## Example

### ❌ Before (Scattered Registration)

```csharp
// ❌ Domain layer knows about DI
using Microsoft.Extensions.DependencyInjection;  // ❌ Wrong

namespace MyApp.Domain
{
    public static class DomainServiceCollectionExtensions
    {
        public static void AddDomain(this IServiceCollection services)  // ❌ Domain depends on DI
        {
            services.AddScoped<PricingService>();
        }
    }
}

// ❌ Application layer registers infrastructure
namespace MyApp.Application
{
    public static class ApplicationServiceCollectionExtensions
    {
        public static void AddApplication(this IServiceCollection services)
        {
            services.AddScoped<IOrderRepository, SqlOrderRepository>();  // ❌ Application knows about SQL
        }
    }
}
```

### ✅ After (Centralized Composition Root)

```csharp
// Domain/DomainAssemblyMarker.cs (just a marker, no DI knowledge)
namespace MyApp.Domain
{
    /// <summary>
    /// Marker class for assembly scanning.
    /// Domain has NO knowledge of DI frameworks.
    /// </summary>
    public sealed class DomainAssemblyMarker { }
}

// Application/ApplicationAssemblyMarker.cs (just a marker)
namespace MyApp.Application
{
    public sealed class ApplicationAssemblyMarker { }
}

// Infrastructure/DependencyInjection/InfrastructureModule.cs
namespace MyApp.Infrastructure.DependencyInjection
{
    /// <summary>
    /// Infrastructure layer exposes extension method for its own registrations.
    /// </summary>
    public static class InfrastructureModule
    {
        public static IServiceCollection AddInfrastructure(
            this IServiceCollection services,
            IConfiguration configuration)
        {
            // ✅ Database
            var connectionString = configuration.GetConnectionString("DefaultConnection");
            services.AddDbContext<MyDbContext>(options =>
                options.UseSqlServer(connectionString));
            
            // ✅ Repositories (implements domain ports)
            services.AddScoped<IOrderRepository, SqlOrderRepository>();
            services.AddScoped<IProductRepository, SqlProductRepository>();
            services.AddScoped<ICustomerRepository, SqlCustomerRepository>();
            
            // ✅ Unit of Work
            services.AddScoped<IUnitOfWork, EfCoreUnitOfWork>();
            
            // ✅ External services (implements application ports)
            var emailProvider = configuration.GetValue<string>("EmailProvider");
            if (emailProvider == "SendGrid")
            {
                services.AddScoped<IEmailService, SendGridEmailAdapter>();
                services.AddSingleton<ISendGridClient>(sp =>
                {
                    var apiKey = configuration.GetValue<string>("SendGrid:ApiKey");
                    return new SendGridClient(apiKey);
                });
            }
            else
            {
                services.AddScoped<IEmailService, ConsoleEmailAdapter>();
            }
            
            // ✅ Payment gateway
            services.AddScoped<IPaymentGateway, StripePaymentAdapter>();
            
            return services;
        }
    }
}

// WebApi/Program.cs (Composition Root)
namespace MyApp.WebApi
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var builder = WebApplication.CreateBuilder(args);
            
            // ✅ Composition Root - all wiring happens here
            ConfigureServices(builder.Services, builder.Configuration);
            
            var app = builder.Build();
            
            // Configure middleware pipeline
            if (app.Environment.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }
            
            app.UseHttpsRedirection();
            app.UseAuthorization();
            app.MapControllers();
            
            app.Run();
        }
        
        private static void ConfigureServices(
            IServiceCollection services,
            IConfiguration configuration)
        {
            // ✅ Controllers
            services.AddControllers();
            
            // ✅ Domain Services (scan assembly for services)
            services.Scan(scan => scan
                .FromAssemblyOf<DomainAssemblyMarker>()
                .AddClasses(classes => classes.Where(t => 
                    t.Name.EndsWith("Service") && 
                    t.Namespace.StartsWith("MyApp.Domain.Services")))
                .AsSelf()
                .WithScopedLifetime());
            
            // ✅ Command Handlers (scan for all handlers)
            services.Scan(scan => scan
                .FromAssemblyOf<ApplicationAssemblyMarker>()
                .AddClasses(classes => classes.AssignableTo(typeof(ICommandHandler<,,>)))
                .AsSelfWithInterfaces()
                .WithScopedLifetime());
            
            // ✅ Query Handlers
            services.Scan(scan => scan
                .FromAssemblyOf<ApplicationAssemblyMarker>()
                .AddClasses(classes => classes.AssignableTo(typeof(IQueryHandler<,>)))
                .AsSelfWithInterfaces()
                .WithScopedLifetime());
            
            // ✅ Validators (FluentValidation)
            services.AddValidatorsFromAssemblyContaining<ApplicationAssemblyMarker>();
            
            // ✅ Infrastructure (repositories, external services)
            services.AddInfrastructure(configuration);
            
            // ✅ Decorators (cross-cutting concerns)
            services.Decorate(typeof(ICommandHandler<,,>), typeof(LoggingCommandHandlerDecorator<,,>));
            services.Decorate(typeof(ICommandHandler<,,>), typeof(ValidationCommandHandlerDecorator<,,>));
            services.Decorate(typeof(ICommandHandler<,,>), typeof(TransactionCommandHandlerDecorator<,,>));
            
            // ✅ Event Dispatcher
            services.AddScoped<IDomainEventDispatcher, DomainEventDispatcher>();
            
            // ✅ Event Handlers (scan for all event handlers)
            services.Scan(scan => scan
                .FromAssemblyOf<ApplicationAssemblyMarker>()
                .AddClasses(classes => classes.AssignableTo(typeof(IDomainEventHandler<>)))
                .AsImplementedInterfaces()
                .WithScopedLifetime());
            
            // ✅ Current User Service
            services.AddScoped<ICurrentUserService, HttpContextCurrentUserService>();
            
            // ✅ Logging
            services.AddLogging(logging =>
            {
                logging.AddConsole();
                logging.AddDebug();
            });
        }
    }
}
```

## Advanced: Multi-Project Configuration

For larger applications with multiple entry points (Web API, Console App, Background Worker):

```csharp
// Shared/DependencyInjection/ServiceCollectionExtensions.cs
namespace MyApp.Shared.DependencyInjection
{
    /// <summary>
    /// Shared registration logic used by all composition roots.
    /// </summary>
    public static class ServiceCollectionExtensions
    {
        public static IServiceCollection AddApplicationCore(
            this IServiceCollection services,
            IConfiguration configuration)
        {
            // Domain services
            services.AddDomainServices();
            
            // Application services
            services.AddCommandHandlers();
            services.AddQueryHandlers();
            services.AddEventHandlers();
            
            // Infrastructure
            services.AddInfrastructure(configuration);
            
            return services;
        }
        
        private static IServiceCollection AddDomainServices(
            this IServiceCollection services)
        {
            services.AddScoped<PricingService>();
            services.AddScoped<FraudDetectionService>();
            return services;
        }
        
        private static IServiceCollection AddCommandHandlers(
            this IServiceCollection services)
        {
            services.Scan(scan => scan
                .FromAssemblyOf<ApplicationAssemblyMarker>()
                .AddClasses(classes => classes.AssignableTo(typeof(ICommandHandler<,,>)))
                .AsSelfWithInterfaces()
                .WithScopedLifetime());
            
            return services;
        }
        
        private static IServiceCollection AddQueryHandlers(
            this IServiceCollection services)
        {
            services.Scan(scan => scan
                .FromAssemblyOf<ApplicationAssemblyMarker>()
                .AddClasses(classes => classes.AssignableTo(typeof(IQueryHandler<,>)))
                .AsSelfWithInterfaces()
                .WithScopedLifetime());
            
            return services;
        }
        
        private static IServiceCollection AddEventHandlers(
            this IServiceCollection services)
        {
            services.AddScoped<IDomainEventDispatcher, DomainEventDispatcher>();
            
            services.Scan(scan => scan
                .FromAssemblyOf<ApplicationAssemblyMarker>()
                .AddClasses(classes => classes.AssignableTo(typeof(IDomainEventHandler<>)))
                .AsImplementedInterfaces()
                .WithScopedLifetime());
            
            return services;
        }
    }
}

// WebApi/Program.cs (Web API Composition Root)
public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);
        
        // ✅ Web-specific services
        builder.Services.AddControllers();
        builder.Services.AddHttpContextAccessor();
        builder.Services.AddScoped<ICurrentUserService, HttpContextCurrentUserService>();
        
        // ✅ Shared core
        builder.Services.AddApplicationCore(builder.Configuration);
        
        var app = builder.Build();
        app.Run();
    }
}

// Console/Program.cs (Console App Composition Root)
public class Program
{
    public static async Task<int> Main(string[] args)
    {
        var configuration = new ConfigurationBuilder()
            .AddJsonFile("appsettings.json")
            .Build();
        
        var services = new ServiceCollection();
        
        // ✅ Console-specific services
        services.AddLogging(logging => logging.AddConsole());
        
        // ✅ Shared core
        services.AddApplicationCore(configuration);
        
        var serviceProvider = services.BuildServiceProvider();
        
        var commandHandler = serviceProvider.GetRequiredService<PlaceOrderCommandHandler>();
        
        // Execute command
        var command = ParseCommandLineArgs(args);
        var result = await commandHandler.HandleAsync(command);
        
        return result.Match(
            onSuccess: _ => 0,
            onFailure: _ => 1);
    }
}
```

## Benefits

1. **Single Location**: All wiring in one place
2. **Framework Agnostic**: Core doesn't know about DI
3. **Testability**: Easy to create test composition roots
4. **Maintainability**: Changes to dependencies centralized
5. **Clarity**: Entire object graph visible in one place

## Testing Composition Root

```csharp
// Tests/Integration/CompositionRootTests.cs
public class CompositionRootTests
{
    [Fact]
    public void CompositionRoot_ResolvesAllRegisteredServices()
    {
        var services = new ServiceCollection();
        var configuration = new ConfigurationBuilder().Build();
        
        // Use production composition root
        services.AddApplicationCore(configuration);
        
        var serviceProvider = services.BuildServiceProvider();
        
        // Verify all services can be resolved
        Assert.NotNull(serviceProvider.GetService<PlaceOrderCommandHandler>());
        Assert.NotNull(serviceProvider.GetService<IOrderRepository>());
        Assert.NotNull(serviceProvider.GetService<PricingService>());
    }
}
```

## See Also

- [Dependency Injection](./dependency-injection.md) — DI pattern
- [Clean Architecture](./clean-architecture.md) — layer organization
- [Infrastructure Layer Organization](./infrastructure-layer-organization.md) — structuring infrastructure
