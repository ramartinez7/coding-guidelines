# Assemblies vs. Namespaces for Code Isolation

> Using namespaces alone for code organization when assemblies provide actual enforcement—use separate assemblies to create real boundaries with `internal` visibility.

## Problem

Namespaces are organizational—they group related code but don't enforce boundaries. Any code can reference any public type in any namespace. Assemblies provide actual isolation: `internal` members are invisible across assembly boundaries.

## The Key Difference

| Mechanism | Enforced By | `internal` Visibility |
|-----------|-------------|----------------------|
| Namespace | Convention only | All code in the solution can see `internal` |
| Assembly | Compiler | Only code in the same assembly can see `internal` |

## Example

### ❌ Namespace-Only Isolation

```
MyApp/
├── Domain/
│   └── Order.cs           (namespace MyApp.Domain)
├── Infrastructure/
│   └── OrderRepository.cs (namespace MyApp.Infrastructure)
└── Api/
    └── OrderController.cs (namespace MyApp.Api)
```

```csharp
// Domain/Order.cs
namespace MyApp.Domain;

public class Order
{
    // "internal" means nothing—everyone in the solution can see this
    internal void BypassValidation() { /* ... */ }
}

// Api/OrderController.cs
namespace MyApp.Api;

public class OrderController
{
    public void Hack()
    {
        var order = new Order();
        order.BypassValidation();  // ✅ Compiles! Same assembly, different namespace
    }
}
```

The namespace boundary is purely organizational. Any code can reach across it.

### ✅ Assembly-Based Isolation

```
Solution/
├── MyApp.Domain/          (separate .csproj)
│   └── Order.cs
├── MyApp.Infrastructure/  (separate .csproj, references Domain)
│   └── OrderRepository.cs
└── MyApp.Api/             (separate .csproj, references Domain & Infrastructure)
    └── OrderController.cs
```

```csharp
// MyApp.Domain/Order.cs
namespace MyApp.Domain;

public class Order
{
    // Truly internal—only visible within MyApp.Domain.dll
    internal void BypassValidation() { /* ... */ }
    
    // Public API is the only way in from other assemblies
    public static Result<Order, OrderError> Create(/* ... */) { /* ... */ }
}

// MyApp.Api/OrderController.cs
namespace MyApp.Api;

public class OrderController
{
    public void Hack()
    {
        var order = new Order();
        order.BypassValidation();  // ❌ Compile error! Not visible across assemblies
    }
}
```

## When to Use Each

### Use Namespaces For

- Logical grouping within a single concern
- Avoiding name collisions
- Organizing code for discoverability
- Small projects where overhead of multiple assemblies isn't worth it

```csharp
// Fine as namespaces within a single Domain assembly
namespace MyApp.Domain.Orders;
namespace MyApp.Domain.Customers;
namespace MyApp.Domain.Products;
```

### Use Separate Assemblies For

- **Enforcing architectural boundaries**: Domain can't depend on Infrastructure
- **Protecting internal implementation**: `internal` actually means internal
- **Capability security**: `internal` constructors on capability tokens
- **Independent deployment**: NuGet packages, microservices
- **Build performance**: Parallel compilation, incremental builds

## The `internal` Constructor Pattern

This pattern from [Capability Security](./capability-security.md) and [Static Factory Methods](./static-factory-methods.md) **requires** assembly boundaries:

```csharp
// MyApp.Domain/Capability.cs
public sealed class Capability<TAction> where TAction : IRequiresAuthorization
{
    public UserId AuthorizedUser { get; }
    
    // Only code in MyApp.Domain can create this
    internal Capability(UserId user) => AuthorizedUser = user;
}

// MyApp.Domain/AuthorizationService.cs
public class AuthorizationService
{
    public Option<Capability<TAction>> Authorize<TAction>(UserId user, TAction action)
        where TAction : IRequiresAuthorization
    {
        if (!IsAuthorized(user, action))
            return Option<Capability<TAction>>.None;
        
        // We're in the same assembly—can call internal constructor
        return Option<Capability<TAction>>.Some(new Capability<TAction>(user));
    }
}
```

If `AuthorizationService` and `Capability` were in the same namespace but different assemblies, this wouldn't work. If they're in the same assembly but different namespaces, it works fine.

## Recommended Project Structure

```
Solution/
├── MyApp.Domain/
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Capabilities/
│   └── Services/           (domain services only)
│
├── MyApp.Application/      (references Domain)
│   ├── Commands/
│   ├── Queries/
│   └── Handlers/
│
├── MyApp.Infrastructure/   (references Domain, Application)
│   ├── Persistence/
│   ├── ExternalServices/
│   └── Messaging/
│
└── MyApp.Api/              (references all above)
    ├── Controllers/
    └── Middleware/
```

### Dependency Rules (Enforced by Assembly References)

```
Domain ← Application ← Infrastructure ← Api
   ↑                           ↑
   └───────────────────────────┘
   
Domain has NO dependencies on other layers
Application depends only on Domain
Infrastructure implements Application interfaces
Api wires everything together
```

These dependencies are enforced by `.csproj` references—you literally cannot violate them.

## InternalsVisibleTo for Testing

Allow test assemblies to see internals:

```csharp
// In MyApp.Domain/AssemblyInfo.cs or .csproj
[assembly: InternalsVisibleTo("MyApp.Domain.Tests")]
```

Or in `.csproj`:

```xml
<ItemGroup>
  <AssemblyAttribute Include="System.Runtime.CompilerServices.InternalsVisibleToAttribute">
    <_Parameter1>MyApp.Domain.Tests</_Parameter1>
  </AssemblyAttribute>
</ItemGroup>
```

This lets tests verify internal behavior without exposing it to production code.

## Trade-offs

### Assembly Isolation

**Pros:**
- Compiler-enforced boundaries
- True `internal` visibility
- Independent versioning and deployment
- Parallel builds
- Clear dependency graph

**Cons:**
- More `.csproj` files to manage
- Cross-assembly calls have (tiny) overhead
- More ceremony for small projects
- Need to publish types to share them

### Namespace-Only

**Pros:**
- Simpler project structure
- Faster iteration in small projects
- No cross-assembly concerns
- Easy refactoring

**Cons:**
- `internal` doesn't enforce anything
- Architectural boundaries are suggestions
- Easy to accidentally create circular dependencies
- Capabilities can be forged

## The Hybrid Approach

Use namespaces within assemblies, assemblies between architectural layers:

```
MyApp.Domain/                    # Assembly boundary
├── Orders/                      # Namespace grouping
│   ├── Order.cs
│   ├── OrderLine.cs
│   └── OrderStatus.cs
├── Customers/                   # Namespace grouping
│   ├── Customer.cs
│   └── CustomerType.cs
└── Shared/                      # Namespace grouping
    ├── Money.cs
    └── DateRange.cs
```

## Benefits

- **Enforced architecture**: Dependencies are compile-time checked
- **True encapsulation**: `internal` means invisible outside the assembly
- **Security guarantees**: Capability tokens can't be forged from other assemblies
- **Clear contracts**: Public API is the only way to interact across boundaries
- **Testable isolation**: `InternalsVisibleTo` for tests only

## See Also

- [Capability Security](./capability-security.md) — requires `internal` constructors
- [Static Factory Methods](./static-factory-methods.md) — private constructors, public factories
- [Nullable Reference Types](./nullable-reference-types.md) — project-level configuration
