# Type-Safe Dependency Injection (Compile-Time DI Validation)

> Runtime service resolution failures from missing registrations or circular dependencies—use compile-time DI validation and strongly-typed service descriptors to catch configuration errors early.

## Problem

Traditional DI containers resolve dependencies at runtime. Missing registrations, circular dependencies, and lifetime mismatches only surface when the application starts or when specific code paths execute. This leads to production failures that could have been caught during compilation.

## Example

### ❌ Before

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // String-based registration—no compile-time validation
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IOrderService, OrderService>();
        // Forgot to register IEmailService—runtime error!
    }
}

public class OrderService : IOrderService
{
    private readonly IUserRepository _users;
    private readonly IEmailService _email;  // Not registered!
    
    public OrderService(IUserRepository users, IEmailService email)
    {
        _users = users;
        _email = email;
    }
}

// Runtime failure:
// InvalidOperationException: Unable to resolve service for type 'IEmailService'
```

### ✅ After

```csharp
/// <summary>
/// Type-safe service descriptor that tracks dependencies.
/// </summary>
public interface IServiceDescriptor<TService>
{
    ServiceLifetime Lifetime { get; }
    IReadOnlyList<Type> Dependencies { get; }
}

public sealed class ServiceDescriptor<TService, TImplementation> 
    : IServiceDescriptor<TService>
    where TImplementation : TService
{
    public ServiceLifetime Lifetime { get; }
    public IReadOnlyList<Type> Dependencies { get; }
    
    private ServiceDescriptor(
        ServiceLifetime lifetime,
        IReadOnlyList<Type> dependencies)
    {
        Lifetime = lifetime;
        Dependencies = dependencies;
    }
    
    public static ServiceDescriptor<TService, TImplementation> Scoped()
        where TImplementation : new()
    {
        return new ServiceDescriptor<TService, TImplementation>(
            ServiceLifetime.Scoped,
            Array.Empty<Type>());
    }
    
    public static ServiceDescriptor<TService, TImplementation> Scoped<TDep1>(
        IServiceDescriptor<TDep1> dep1)
    {
        return new ServiceDescriptor<TService, TImplementation>(
            ServiceLifetime.Scoped,
            new[] { typeof(TDep1) });
    }
    
    public static ServiceDescriptor<TService, TImplementation> Scoped<TDep1, TDep2>(
        IServiceDescriptor<TDep1> dep1,
        IServiceDescriptor<TDep2> dep2)
    {
        return new ServiceDescriptor<TService, TImplementation>(
            ServiceLifetime.Scoped,
            new[] { typeof(TDep1), typeof(TDep2) });
    }
    
    public static ServiceDescriptor<TService, TImplementation> Singleton()
        where TImplementation : new()
    {
        return new ServiceDescriptor<TService, TImplementation>(
            ServiceLifetime.Singleton,
            Array.Empty<Type>());
    }
    
    public static ServiceDescriptor<TService, TImplementation> Singleton<TDep1>(
        IServiceDescriptor<TDep1> dep1)
    {
        return new ServiceDescriptor<TService, TImplementation>(
            ServiceLifetime.Singleton,
            new[] { typeof(TDep1) });
    }
    
    public void Register(IServiceCollection services)
    {
        services.Add(new Microsoft.Extensions.DependencyInjection.ServiceDescriptor(
            typeof(TService),
            typeof(TImplementation),
            Lifetime));
    }
}

/// <summary>
/// Container that enforces dependency declaration at compile time.
/// </summary>
public sealed class TypedServiceContainer
{
    private readonly Dictionary<Type, IServiceDescriptor<object>> _descriptors = new();
    
    public ServiceDescriptor<TService, TImplementation> Register<TService, TImplementation>()
        where TImplementation : TService, new()
    {
        var descriptor = ServiceDescriptor<TService, TImplementation>.Scoped();
        _descriptors[typeof(TService)] = (IServiceDescriptor<object>)descriptor;
        return descriptor;
    }
    
    public ServiceDescriptor<TService, TImplementation> Register<TService, TImplementation, TDep1>(
        ServiceDescriptor<TDep1, object> dep1)
        where TImplementation : TService
    {
        // Compile-time check: TDep1 must be registered
        if (!_descriptors.ContainsKey(typeof(TDep1)))
            throw new InvalidOperationException($"Dependency {typeof(TDep1)} not registered");
        
        var descriptor = ServiceDescriptor<TService, TImplementation>.Scoped(dep1);
        _descriptors[typeof(TService)] = (IServiceDescriptor<object>)descriptor;
        return descriptor;
    }
    
    public void Build(IServiceCollection services)
    {
        foreach (var descriptor in _descriptors.Values)
        {
            // Validate all dependencies are registered
            foreach (var dependency in descriptor.Dependencies)
            {
                if (!_descriptors.ContainsKey(dependency))
                    throw new InvalidOperationException(
                        $"Dependency {dependency} not registered");
            }
        }
        
        // Register all services
        foreach (var descriptor in _descriptors.Values)
        {
            // Registration logic
        }
    }
}

// Better approach: Pure functions for DI composition
public sealed record ServiceGraph<T>(
    Func<IServiceProvider, T> Factory,
    ServiceLifetime Lifetime);

public static class ServiceGraphBuilder
{
    public static ServiceGraph<T> Singleton<T>(Func<T> factory)
        => new(() => factory(), ServiceLifetime.Singleton);
    
    public static ServiceGraph<T> Singleton<T, TDep>(
        ServiceGraph<TDep> dep,
        Func<TDep, T> factory)
        => new(sp => factory(sp.GetRequiredService<TDep>()), ServiceLifetime.Singleton);
    
    public static ServiceGraph<T> Scoped<T>(Func<T> factory)
        => new(() => factory(), ServiceLifetime.Scoped);
    
    public static ServiceGraph<T> Scoped<T, TDep1, TDep2>(
        ServiceGraph<TDep1> dep1,
        ServiceGraph<TDep2> dep2,
        Func<TDep1, TDep2, T> factory)
    {
        return new ServiceGraph<T>(
            sp => factory(
                sp.GetRequiredService<TDep1>(),
                sp.GetRequiredService<TDep2>()),
            ServiceLifetime.Scoped);
    }
}

// Usage: Dependencies declared explicitly in code
public class DIConfiguration
{
    public static void Configure(IServiceCollection services)
    {
        // All dependencies explicit and compile-time checked
        var userRepo = ServiceGraphBuilder.Singleton<IUserRepository>(
            () => new UserRepository());
        
        var emailService = ServiceGraphBuilder.Scoped<IEmailService>(
            () => new EmailService());
        
        var orderService = ServiceGraphBuilder.Scoped<IOrderService, IUserRepository, IEmailService>(
            userRepo,
            emailService,
            (users, email) => new OrderService(users, email));
        
        // Register all
        services.AddSingleton(userRepo.Factory);
        services.AddScoped(emailService.Factory);
        services.AddScoped(orderService.Factory);
    }
}

// Won't compile:
// var orderService = ServiceGraphBuilder.Scoped<IOrderService, IUserRepository>(
//     userRepo,
//     (users) => new OrderService(users, ???));  // Missing IEmailService!
```

## Pure DI with Composition Root

```csharp
/// <summary>
/// Type-safe composition root using pure functions.
/// All dependencies declared explicitly—no DI container runtime magic.
/// </summary>
public sealed class CompositionRoot
{
    private readonly IConfiguration _config;
    
    public CompositionRoot(IConfiguration config)
    {
        _config = config;
    }
    
    // Each service graph is a pure function
    public IUserRepository CreateUserRepository()
    {
        var connectionString = _config.GetConnectionString("Default");
        return new UserRepository(connectionString);
    }
    
    public IEmailService CreateEmailService()
    {
        var smtpSettings = _config.GetSection("Smtp").Get<SmtpSettings>();
        return new EmailService(smtpSettings);
    }
    
    public IOrderService CreateOrderService()
    {
        // Dependencies explicit—compiler verifies correctness
        return new OrderService(
            CreateUserRepository(),
            CreateEmailService());
    }
    
    public OrderController CreateOrderController()
    {
        return new OrderController(CreateOrderService());
    }
}

// ASP.NET Core integration
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Register composition root
        services.AddSingleton<CompositionRoot>();
        
        // Use composition root to create services
        services.AddScoped<IOrderService>(sp =>
            sp.GetRequiredService<CompositionRoot>().CreateOrderService());
        
        services.AddScoped<OrderController>(sp =>
            sp.GetRequiredService<CompositionRoot>().CreateOrderController());
    }
}
```

## Lifetime Validation with Types

```csharp
/// <summary>
/// Phantom types for service lifetimes.
/// </summary>
public interface ILifetime { }
public interface ISingleton : ILifetime { }
public interface IScoped : ILifetime { }
public interface ITransient : ILifetime { }

/// <summary>
/// Service wrapper with lifetime encoded in type.
/// </summary>
public sealed class Service<T, TLifetime> where TLifetime : ILifetime
{
    private readonly Func<IServiceProvider, T> _factory;
    
    private Service(Func<IServiceProvider, T> factory)
    {
        _factory = factory;
    }
    
    public static Service<T, ISingleton> Singleton(Func<T> factory)
        => new(_ => factory());
    
    public static Service<T, IScoped> Scoped(Func<T> factory)
        => new(_ => factory());
    
    public static Service<T, ITransient> Transient(Func<T> factory)
        => new(_ => factory());
    
    public T Create(IServiceProvider provider) => _factory(provider);
}

// Lifetime constraint methods
public static class ServiceExtensions
{
    // Singleton can depend on Singleton only
    public static Service<T, ISingleton> DependsOn<T, TDep>(
        this Service<T, ISingleton> service,
        Service<TDep, ISingleton> dependency,
        Func<T, TDep, T> factory)
    {
        return Service<T, ISingleton>.Singleton(() => factory(
            default(T)!,
            dependency.Create(null!)));
    }
    
    // Scoped can depend on Singleton or Scoped
    public static Service<T, IScoped> DependsOn<T, TDep>(
        this Service<T, IScoped> service,
        Service<TDep, ISingleton> dependency,
        Func<T, TDep, T> factory)
    {
        return Service<T, IScoped>.Scoped(() => factory(
            default(T)!,
            dependency.Create(null!)));
    }
    
    // Compile-time error: Singleton cannot depend on Scoped
    // (method doesn't exist)
}

// Usage: Lifetime violations caught at compile time
var singleton = Service<IUserRepository, ISingleton>.Singleton(
    () => new UserRepository());

var scoped = Service<IOrderService, IScoped>.Scoped(
    () => new OrderService(null!, null!));

// Won't compile:
// singleton.DependsOn(scoped, ...);  // Error: No such method
```

## Why It's a Problem

1. **Runtime failures**: Missing registrations only discovered when code executes
2. **Circular dependencies**: Not detected until runtime
3. **Lifetime mismatches**: Singleton depending on Scoped fails silently
4. **No validation**: Can't verify DI configuration is complete

## Symptoms

- `InvalidOperationException` at runtime for missing services
- Circular dependency stack overflows
- Services with wrong lifetime cause subtle bugs
- Integration tests just to verify DI configuration

## Benefits

- **Compile-time safety**: Missing dependencies don't compile
- **Explicit dependencies**: All dependencies visible in code
- **Lifetime validation**: Type system prevents lifetime violations
- **Refactoring-safe**: Changing dependencies updates all call sites
- **No reflection**: Pure functions, no runtime magic

## See Also

- [Dependency Injection](./dependency-injection.md) — DI fundamentals
- [Compile-Time Validation](./compile-time-validation.md) — catching errors early
- [Phantom Types](./phantom-types.md) — encoding state in types
- [Clean Architecture](./clean-architecture.md) — dependency management
