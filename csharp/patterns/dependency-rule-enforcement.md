# Dependency Rule Enforcement (Architectural Constraints)

> Dependencies between layers can accidentally point the wrong direction—use compile-time checks to enforce that dependencies flow inward toward the domain.

## Problem

In clean architecture, the dependency rule states that source code dependencies must point inward: Domain depends on nothing, Application depends only on Domain, Infrastructure depends on Application and Domain. However, without enforcement, developers can accidentally create backwards dependencies that violate this rule.

## The Dependency Rule

```
┌─────────────────────────────────────┐
│    Infrastructure Layer             │  ← Can depend on Application + Domain
│  ┌─────────────────────────────┐    │
│  │    Application Layer        │    │  ← Can depend on Domain only
│  │  ┌─────────────────────┐    │    │
│  │  │   Domain Layer      │    │    │  ← Depends on NOTHING
│  │  │                     │    │    │
│  │  └─────────────────────┘    │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘

✓ Infrastructure → Application → Domain
✗ Domain → Application
✗ Application → Infrastructure
✗ Domain → Infrastructure
```

## Example

### ❌ Before (No Enforcement)

```csharp
// Domain/Entities/Order.cs
using MyApp.Infrastructure.Persistence;  // ❌ Domain depends on Infrastructure!
using Microsoft.EntityFrameworkCore;     // ❌ Domain depends on EF!

namespace MyApp.Domain.Entities
{
    public class Order
    {
        public OrderId Id { get; set; }
        
        // ❌ Domain entity knows about database
        public void SaveToDatabase(MyDbContext dbContext)
        {
            dbContext.Orders.Add(this);
            dbContext.SaveChanges();
        }
    }
}

// Application/UseCases/PlaceOrderUseCase.cs
using MyApp.Infrastructure.Email;        // ❌ Application depends on Infrastructure!

namespace MyApp.Application.UseCases
{
    public class PlaceOrderUseCase
    {
        private readonly SendGridEmailService _emailService;  // ❌ Concrete implementation!
        
        public PlaceOrderUseCase(SendGridEmailService emailService)
        {
            _emailService = emailService;
        }
    }
}
```

**Problems:**
- Domain layer has dependencies on Infrastructure
- Application layer depends on Infrastructure implementations
- Can't test Domain without database
- Can't swap email provider without changing Application layer
- Circular dependencies possible

### ✅ After (Enforced Dependencies)

#### 1. Define Dependencies in Project Files

```xml
<!-- Domain/Domain.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  
  <!-- Domain has NO dependencies on other projects -->
  <!-- This is enforced at compile time -->
</Project>

<!-- Application/Application.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  
  <ItemGroup>
    <!-- Application can only depend on Domain -->
    <ProjectReference Include="..\Domain\Domain.csproj" />
  </ItemGroup>
  
  <!-- ❌ Cannot reference Infrastructure - compiler will reject -->
</Project>

<!-- Infrastructure/Infrastructure.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  
  <ItemGroup>
    <!-- Infrastructure can depend on both -->
    <ProjectReference Include="..\Domain\Domain.csproj" />
    <ProjectReference Include="..\Application\Application.csproj" />
    
    <!-- Infrastructure can use external packages -->
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
    <PackageReference Include="SendGrid" Version="9.28.0" />
  </ItemGroup>
</Project>

<!-- WebApi/WebApi.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  
  <ItemGroup>
    <!-- WebApi needs Application (for use cases) and Infrastructure (for DI setup) -->
    <ProjectReference Include="..\Application\Application.csproj" />
    <ProjectReference Include="..\Infrastructure\Infrastructure.csproj" />
    
    <!-- WebApi does NOT reference Domain directly (goes through Application) -->
  </ItemGroup>
</Project>
```

#### 2. Use Dependency Inversion Principle

```csharp
// Domain/Interfaces/IOrderRepository.cs
namespace MyApp.Domain.Interfaces
{
    /// <summary>
    /// Interface defined in Domain layer.
    /// Implementation will be in Infrastructure, but Domain doesn't know that.
    /// </summary>
    public interface IOrderRepository
    {
        Task<Option<Order>> GetByIdAsync(OrderId id);
        Task SaveAsync(Order order);
    }
}

// Domain/Entities/Order.cs
namespace MyApp.Domain.Entities
{
    /// <summary>
    /// Domain entity with no infrastructure dependencies.
    /// Pure business logic.
    /// </summary>
    public sealed class Order
    {
        private readonly List<OrderItem> _items = new();
        
        public OrderId Id { get; }
        public Money Total { get; private set; }
        
        // ✓ Business logic only - no database, no HTTP, no frameworks
        public Result<Unit, string> AddItem(Product product, int quantity)
        {
            if (quantity <= 0)
                return Result<Unit, string>.Failure("Quantity must be positive");
            
            var item = new OrderItem(product.Id, quantity, product.Price);
            _items.Add(item);
            RecalculateTotal();
            
            return Result<Unit, string>.Success(Unit.Value);
        }
        
        private void RecalculateTotal()
        {
            Total = _items.Select(i => i.Total).Aggregate(Money.Zero, (a, b) => a + b);
        }
    }
}

// Application/Interfaces/IEmailService.cs
namespace MyApp.Application.Interfaces
{
    /// <summary>
    /// Interface defined in Application layer.
    /// Implementation will be in Infrastructure.
    /// </summary>
    public interface IEmailService
    {
        Task SendOrderConfirmationAsync(OrderId orderId, Email customerEmail);
    }
}

// Application/UseCases/PlaceOrderUseCase.cs
namespace MyApp.Application.UseCases
{
    /// <summary>
    /// Use case depends on abstractions (interfaces) from Domain and Application.
    /// No dependency on Infrastructure layer.
    /// </summary>
    public sealed class PlaceOrderUseCase
    {
        private readonly IOrderRepository _orderRepository;        // From Domain
        private readonly IProductRepository _productRepository;    // From Domain
        private readonly IEmailService _emailService;              // From Application
        private readonly IUnitOfWork _unitOfWork;                  // From Application
        
        public PlaceOrderUseCase(
            IOrderRepository orderRepository,
            IProductRepository productRepository,
            IEmailService emailService,
            IUnitOfWork unitOfWork)
        {
            _orderRepository = orderRepository;
            _productRepository = productRepository;
            _emailService = emailService;
            _unitOfWork = unitOfWork;
        }
        
        public async Task<Result<OrderId, PlaceOrderError>> ExecuteAsync(
            PlaceOrderCommand command)
        {
            // Orchestrate domain logic
            var order = Order.Create(command.CustomerId);
            
            foreach (var item in command.Items)
            {
                var productOpt = await _productRepository.GetByIdAsync(item.ProductId);
                // ... add items to order
            }
            
            await _orderRepository.SaveAsync(order);
            await _unitOfWork.CommitAsync();
            
            // Send email via abstraction
            await _emailService.SendOrderConfirmationAsync(order.Id, command.CustomerEmail);
            
            return Result<OrderId, PlaceOrderError>.Success(order.Id);
        }
    }
}

// Infrastructure/Persistence/OrderRepository.cs
namespace MyApp.Infrastructure.Persistence
{
    using MyApp.Domain.Interfaces;    // ✓ Infrastructure depends on Domain
    using MyApp.Domain.Entities;      // ✓ Infrastructure depends on Domain
    
    /// <summary>
    /// Implementation of Domain interface using Entity Framework.
    /// Domain knows nothing about this class.
    /// </summary>
    public sealed class OrderRepository : IOrderRepository
    {
        private readonly MyDbContext _dbContext;
        
        public async Task<Option<Order>> GetByIdAsync(OrderId id)
        {
            var entity = await _dbContext.Orders.FindAsync(id.Value);
            // Map to domain model...
        }
        
        public async Task SaveAsync(Order order)
        {
            // Map to persistence model and save...
        }
    }
}

// Infrastructure/Email/SendGridEmailService.cs
namespace MyApp.Infrastructure.Email
{
    using MyApp.Application.Interfaces;  // ✓ Infrastructure depends on Application
    
    /// <summary>
    /// Implementation of Application interface using SendGrid.
    /// Application knows nothing about this class.
    /// </summary>
    public sealed class SendGridEmailService : IEmailService
    {
        private readonly ISendGridClient _client;
        
        public async Task SendOrderConfirmationAsync(OrderId orderId, Email customerEmail)
        {
            // SendGrid-specific implementation
        }
    }
}
```

#### 3. Wire Up Dependencies at Composition Root

```csharp
// WebApi/Program.cs (Composition Root)
var builder = WebApplication.CreateBuilder(args);

// Register Application layer services
builder.Services.AddScoped<PlaceOrderUseCase>();

// Register Infrastructure implementations of Domain interfaces
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<IProductRepository, ProductRepository>();

// Register Infrastructure implementations of Application interfaces
builder.Services.AddScoped<IEmailService, SendGridEmailService>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

var app = builder.Build();
```

## Automated Enforcement with ArchUnit

```csharp
// Tests/ArchitectureTests.cs
using NetArchTest.Rules;

namespace MyApp.Tests
{
    public class ArchitectureTests
    {
        private const string DomainNamespace = "MyApp.Domain";
        private const string ApplicationNamespace = "MyApp.Application";
        private const string InfrastructureNamespace = "MyApp.Infrastructure";
        
        [Fact]
        public void Domain_Should_Not_Have_Dependency_On_Application()
        {
            var result = Types.InAssembly(typeof(Order).Assembly)
                .That()
                .ResideInNamespace(DomainNamespace)
                .ShouldNot()
                .HaveDependencyOn(ApplicationNamespace)
                .GetResult();
            
            Assert.True(result.IsSuccessful, 
                $"Domain layer has dependencies on Application layer: {string.Join(", ", result.FailingTypes)}");
        }
        
        [Fact]
        public void Domain_Should_Not_Have_Dependency_On_Infrastructure()
        {
            var result = Types.InAssembly(typeof(Order).Assembly)
                .That()
                .ResideInNamespace(DomainNamespace)
                .ShouldNot()
                .HaveDependencyOn(InfrastructureNamespace)
                .GetResult();
            
            Assert.True(result.IsSuccessful,
                $"Domain layer has dependencies on Infrastructure layer: {string.Join(", ", result.FailingTypes)}");
        }
        
        [Fact]
        public void Application_Should_Not_Have_Dependency_On_Infrastructure()
        {
            var result = Types.InAssembly(typeof(PlaceOrderUseCase).Assembly)
                .That()
                .ResideInNamespace(ApplicationNamespace)
                .ShouldNot()
                .HaveDependencyOn(InfrastructureNamespace)
                .GetResult();
            
            Assert.True(result.IsSuccessful,
                $"Application layer has dependencies on Infrastructure layer: {string.Join(", ", result.FailingTypes)}");
        }
        
        [Fact]
        public void Domain_Should_Not_Reference_EntityFramework()
        {
            var result = Types.InAssembly(typeof(Order).Assembly)
                .That()
                .ResideInNamespace(DomainNamespace)
                .ShouldNot()
                .HaveDependencyOn("Microsoft.EntityFrameworkCore")
                .GetResult();
            
            Assert.True(result.IsSuccessful,
                $"Domain layer references Entity Framework: {string.Join(", ", result.FailingTypes)}");
        }
        
        [Fact]
        public void Interfaces_In_Domain_Should_Be_Implemented_In_Infrastructure()
        {
            var domainInterfaces = Types.InAssembly(typeof(Order).Assembly)
                .That()
                .ResideInNamespace(DomainNamespace)
                .And()
                .AreInterfaces()
                .GetTypes();
            
            var infrastructureTypes = Types.InAssembly(typeof(OrderRepository).Assembly)
                .That()
                .ResideInNamespace(InfrastructureNamespace)
                .GetTypes();
            
            foreach (var domainInterface in domainInterfaces)
            {
                var hasImplementation = infrastructureTypes.Any(t => 
                    !t.IsAbstract && 
                    !t.IsInterface && 
                    domainInterface.IsAssignableFrom(t));
                
                Assert.True(hasImplementation,
                    $"Domain interface {domainInterface.Name} has no implementation in Infrastructure layer");
            }
        }
        
        [Fact]
        public void Controllers_Should_Not_Reference_Domain_Directly()
        {
            var result = Types.InAssembly(typeof(OrdersController).Assembly)
                .That()
                .ResideInNamespace("MyApp.WebApi.Controllers")
                .ShouldNot()
                .HaveDependencyOn(DomainNamespace)
                .GetResult();
            
            Assert.True(result.IsSuccessful,
                $"Controllers have direct dependencies on Domain layer: {string.Join(", ", result.FailingTypes)}");
        }
    }
}
```

## NuGet Package Reference Constraints

```xml
<!-- Directory.Build.props (at solution root) -->
<Project>
  <PropertyGroup>
    <!-- Enable package reference validation -->
    <EnablePackageValidation>true</EnablePackageValidation>
  </PropertyGroup>
  
  <!-- Domain layer: NO external dependencies allowed -->
  <ItemGroup Condition="'$(MSBuildProjectName)' == 'Domain'">
    <!-- Explicitly block infrastructure packages in Domain -->
    <PackageReference Update="Microsoft.EntityFrameworkCore">
      <PrivateAssets>none</PrivateAssets>
      <ExcludeAssets>all</ExcludeAssets>
    </PackageReference>
    <PackageReference Update="SendGrid">
      <PrivateAssets>none</PrivateAssets>
      <ExcludeAssets>all</ExcludeAssets>
    </PackageReference>
    <PackageReference Update="Dapper">
      <PrivateAssets>none</PrivateAssets>
      <ExcludeAssets>all</ExcludeAssets>
    </PackageReference>
  </ItemGroup>
  
  <!-- Application layer: NO infrastructure packages -->
  <ItemGroup Condition="'$(MSBuildProjectName)' == 'Application'">
    <PackageReference Update="Microsoft.EntityFrameworkCore">
      <PrivateAssets>none</PrivateAssets>
      <ExcludeAssets>all</ExcludeAssets>
    </PackageReference>
  </ItemGroup>
</Project>
```

## InternalsVisibleTo for Testing

```csharp
// Domain/AssemblyInfo.cs
using System.Runtime.CompilerServices;

// Allow test project to access internal members
[assembly: InternalsVisibleTo("MyApp.Domain.Tests")]

// ❌ Do NOT expose internals to Application or Infrastructure
// [assembly: InternalsVisibleTo("MyApp.Application")]  // NO!
// [assembly: InternalsVisibleTo("MyApp.Infrastructure")]  // NO!
```

## Visual Studio Solution Structure

```
Solution: MyApp
│
├── src/
│   ├── Core/
│   │   ├── MyApp.Domain/              ← No dependencies
│   │   └── MyApp.Application/         ← Depends on Domain only
│   │
│   ├── Infrastructure/
│   │   ├── MyApp.Infrastructure/      ← Depends on Domain + Application
│   │   └── MyApp.Persistence/         ← (optional: separate persistence)
│   │
│   └── Presentation/
│       ├── MyApp.WebApi/              ← Depends on Application + Infrastructure
│       └── MyApp.Cli/                 ← Depends on Application + Infrastructure
│
└── tests/
    ├── MyApp.Domain.Tests/            ← Tests Domain
    ├── MyApp.Application.Tests/       ← Tests Application
    └── MyApp.ArchitectureTests/       ← Tests dependency rules
```

## Dependency Flow Example

```
┌──────────────────────────────────────────────────────┐
│ User Request                                         │
└────────────┬─────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────┐
│ WebApi Layer (Presentation)                          │
│  - OrdersController.Post()                           │
│  - Maps HTTP request to Command                      │
└────────────┬─────────────────────────────────────────┘
             │ Calls
             ▼
┌──────────────────────────────────────────────────────┐
│ Application Layer                                    │
│  - PlaceOrderUseCase.ExecuteAsync()                  │
│  - Orchestrates domain objects                       │
│  - Uses: IOrderRepository (Domain interface)         │
│  - Uses: IEmailService (Application interface)       │
└────────────┬─────────────────────────────────────────┘
             │ Uses
             ▼
┌──────────────────────────────────────────────────────┐
│ Domain Layer                                         │
│  - Order.AddItem() (business logic)                  │
│  - Product.ReserveStock() (business logic)           │
│  - IOrderRepository (interface definition)           │
└──────────────────────────────────────────────────────┘
             ▲
             │ Implements
             │
┌──────────────────────────────────────────────────────┐
│ Infrastructure Layer                                 │
│  - OrderRepository : IOrderRepository                │
│  - SendGridEmailService : IEmailService              │
│  - Entity Framework DbContext                        │
└──────────────────────────────────────────────────────┘

Dependencies point: Infrastructure → Application → Domain
Control flow: WebApi → Application → Domain → Infrastructure (via DI)
```

## Why It's a Problem

1. **Circular dependencies**: Layers depend on each other creating cycles
2. **Tight coupling**: Domain coupled to databases, frameworks, external services
3. **Hard to test**: Need infrastructure to test domain logic
4. **Framework lock-in**: Can't swap technologies without rewriting domain
5. **Accidental violations**: No enforcement means rules get broken over time

## Symptoms

- `using` statements in Domain for EF, ASP.NET, or external libraries
- Can't compile Domain project without Infrastructure
- Domain tests require database or external services
- Circular project references
- Changes to infrastructure break domain

## Benefits

- **Compile-time enforcement**: Impossible to violate dependency rule
- **Pure domain**: Business logic free from infrastructure concerns
- **Testable**: Test domain without any infrastructure
- **Flexible**: Swap frameworks, databases, external services easily
- **Maintainable**: Clear, enforced boundaries prevent coupling

## See Also

- [Clean Architecture](./clean-architecture.md) — overall architecture pattern
- [Dependency Injection](./dependency-injection.md) — wiring up dependencies at runtime
- [Repository Pattern](./repository-pattern.md) — abstracting data access
- [Anti-Corruption Layer](./anti-corruption-layer.md) — protecting domain from external systems
- [DTO vs Domain Boundary](./dto-domain-boundary.md) — separating API from domain
