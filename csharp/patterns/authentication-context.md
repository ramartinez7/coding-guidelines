# Authentication Context (Type-Safe Identity)

> Passing user IDs as primitives and checking `HttpContext.User` in business logic—use an authenticated context type that proves identity at compile time.

## Problem

Business logic that needs to know "who is calling" often accesses `HttpContext.User` directly or accepts raw user IDs. This creates tight coupling to HTTP, makes testing difficult, and provides no compile-time guarantee that authentication occurred.

## Example

### ❌ Before

```csharp
public class OrderService
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public void PlaceOrder(PlaceOrderCommand command)
    {
        // Tightly coupled to HTTP context
        var userId = _httpContextAccessor.HttpContext?.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        if (userId == null)
            throw new UnauthorizedException("User not authenticated");
        
        // What if authentication failed but this code runs anyway?
        var order = new Order
        {
            UserId = Guid.Parse(userId),
            Items = command.Items
        };
        
        _repository.Save(order);
    }
    
    public List<Order> GetUserOrders(Guid userId)
    {
        // Is this the authenticated user's ID, or did someone pass an arbitrary ID?
        // Can any authenticated user see any other user's orders?
        return _repository.GetOrders(userId);
    }
}

public class OrderController
{
    public IActionResult PlaceOrder(PlaceOrderCommand command)
    {
        // No way to tell from the signature that this requires authentication
        _orderService.PlaceOrder(command);
        return Ok();
    }
    
    public IActionResult GetOrders(Guid userId)
    {
        // Dangerous: accepts any user ID, no authorization check
        var orders = _orderService.GetUserOrders(userId);
        return Ok(orders);
    }
}
```

**Problems:**
- Business logic depends on HTTP infrastructure
- No compile-time proof authentication occurred
- Raw GUIDs could be any user, not necessarily the authenticated one
- Authorization checks easily forgotten
- Hard to test without mocking `HttpContext`

### ✅ After

```csharp
/// <summary>
/// Proof that a user has been authenticated.
/// Cannot be constructed outside the authentication middleware.
/// </summary>
public sealed record AuthenticatedUser(
    UserId UserId,
    string Username,
    Email Email,
    IReadOnlySet<Role> Roles,
    DateTime AuthenticatedAt)
{
    // Internal constructor—only authentication middleware can create
    internal AuthenticatedUser(
        UserId userId, 
        string username, 
        Email email, 
        IReadOnlySet<Role> roles) 
        : this(userId, username, email, roles, DateTime.UtcNow)
    { }
    
    public bool HasRole(Role role) => Roles.Contains(role);
    
    public bool HasAnyRole(params Role[] roles) => roles.Any(r => Roles.Contains(r));
    
    public bool HasAllRoles(params Role[] roles) => roles.All(r => Roles.Contains(r));
}

public enum Role
{
    User,
    Admin,
    Moderator
}

// Middleware creates the authenticated context
public class AuthenticationMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(HttpContext context)
    {
        var user = context.User;
        
        if (user?.Identity?.IsAuthenticated == true)
        {
            var userId = UserId.Parse(user.FindFirst(ClaimTypes.NameIdentifier)!.Value);
            var username = user.FindFirst(ClaimTypes.Name)!.Value;
            var emailStr = user.FindFirst(ClaimTypes.Email)!.Value;
            var email = Email.Create(emailStr).Value;  // Assuming valid from auth provider
            
            var roles = user.FindAll(ClaimTypes.Role)
                .Select(c => Enum.Parse<Role>(c.Value))
                .ToHashSet();
            
            var authenticatedUser = new AuthenticatedUser(userId, username, email, roles);
            
            // Store in HttpContext.Items for retrieval
            context.Items["AuthenticatedUser"] = authenticatedUser;
        }
        
        await _next(context);
    }
}

// Extension for controllers to access authenticated user
public static class HttpContextExtensions
{
    public static AuthenticatedUser? GetAuthenticatedUser(this HttpContext context)
    {
        return context.Items["AuthenticatedUser"] as AuthenticatedUser;
    }
}

public class OrderService
{
    // Type signature requires authenticated user—no HttpContext dependency
    public void PlaceOrder(AuthenticatedUser user, PlaceOrderCommand command)
    {
        // No need to check—type guarantees authentication
        var order = new Order
        {
            UserId = user.UserId,
            Items = command.Items,
            PlacedBy = user.Username,
            PlacedAt = DateTime.UtcNow
        };
        
        _repository.Save(order);
        _logger.LogInformation($"Order placed by {user.Username}");
    }
    
    // Only returns orders for the authenticated user
    public List<Order> GetMyOrders(AuthenticatedUser user)
    {
        // Impossible to fetch another user's orders—UserId comes from AuthenticatedUser
        return _repository.GetOrdersByUserId(user.UserId);
    }
}

public class OrderController
{
    [Authorize]  // Ensures middleware populates AuthenticatedUser
    public IActionResult PlaceOrder(PlaceOrderCommand command)
    {
        var user = HttpContext.GetAuthenticatedUser();
        
        if (user == null)
            return Unauthorized();
        
        // Type system enforces passing authenticated user
        _orderService.PlaceOrder(user, command);
        return Ok();
    }
    
    [Authorize]
    public IActionResult GetMyOrders()
    {
        var user = HttpContext.GetAuthenticatedUser();
        
        if (user == null)
            return Unauthorized();
        
        // Cannot pass arbitrary user ID—must use authenticated user
        var orders = _orderService.GetMyOrders(user);
        return Ok(orders);
    }
}
```

## Advanced Patterns

### Role-Specific Contexts

```csharp
/// <summary>
/// An authenticated user with Admin role.
/// Cannot be constructed without role verification.
/// </summary>
public sealed record AdminUser(AuthenticatedUser User)
{
    private AdminUser(AuthenticatedUser user) : this() => User = user;
    
    public static Option<AdminUser> From(AuthenticatedUser user)
    {
        return user.HasRole(Role.Admin)
            ? Option<AdminUser>.Some(new AdminUser(user))
            : Option<AdminUser>.None;
    }
    
    public UserId UserId => User.UserId;
    public string Username => User.Username;
}

// Service methods can require specific roles
public class UserManagementService
{
    // Only admins can call this—type proves it
    public void DeleteUser(AdminUser admin, UserId targetUserId)
    {
        _logger.LogWarning($"Admin {admin.Username} deleting user {targetUserId}");
        _repository.Delete(targetUserId);
    }
}

// Controller
public class AdminController
{
    [Authorize(Roles = "Admin")]
    public IActionResult DeleteUser(Guid userId)
    {
        var user = HttpContext.GetAuthenticatedUser();
        
        if (user == null)
            return Unauthorized();
        
        var adminUser = AdminUser.From(user);
        
        return adminUser.Match(
            onSome: admin =>
            {
                _userService.DeleteUser(admin, new UserId(userId));
                return Ok();
            },
            onNone: () => Forbid()
        );
    }
}
```

### Scope-Based Context

```csharp
/// <summary>
/// User authenticated with specific OAuth scopes.
/// </summary>
public sealed record ScopedUser(
    AuthenticatedUser User,
    IReadOnlySet<string> Scopes)
{
    internal ScopedUser(AuthenticatedUser user, IReadOnlySet<string> scopes) 
        : this() => (User, Scopes) = (user, scopes);
    
    public bool HasScope(string scope) => Scopes.Contains(scope);
}

/// <summary>
/// User authorized to read user data.
/// </summary>
public sealed record ReadUserScope(ScopedUser User)
{
    public static Option<ReadUserScope> From(ScopedUser user)
    {
        return user.HasScope("user:read")
            ? Option<ReadUserScope>.Some(new ReadUserScope(user))
            : Option<ReadUserScope>.None;
    }
}

/// <summary>
/// User authorized to write user data.
/// </summary>
public sealed record WriteUserScope(ScopedUser User)
{
    public static Option<WriteUserScope> From(ScopedUser user)
    {
        return user.HasScope("user:write")
            ? Option<WriteUserScope>.Some(new WriteUserScope(user))
            : Option<WriteUserScope>.None;
    }
}

public class UserService
{
    public UserProfile GetProfile(ReadUserScope scope)
    {
        // Type guarantees user:read scope
        return _repository.GetProfile(scope.User.User.UserId);
    }
    
    public void UpdateProfile(WriteUserScope scope, UpdateProfileCommand command)
    {
        // Type guarantees user:write scope
        var profile = _repository.GetProfile(scope.User.User.UserId);
        profile.Update(command);
        _repository.Save(profile);
    }
}
```

### Anonymous vs Authenticated

```csharp
/// <summary>
/// Represents either an authenticated user or an anonymous visitor.
/// </summary>
public abstract record RequestContext
{
    private RequestContext() { }
    
    public sealed record Anonymous(string IpAddress) : RequestContext;
    public sealed record Authenticated(AuthenticatedUser User) : RequestContext;
}

public class ProductService
{
    public ProductList GetProducts(RequestContext context)
    {
        return context switch
        {
            RequestContext.Authenticated auth => GetProductsForUser(auth.User),
            RequestContext.Anonymous anon => GetPublicProducts(anon.IpAddress),
            _ => throw new UnreachableException()
        };
    }
    
    private ProductList GetProductsForUser(AuthenticatedUser user)
    {
        // Include personalized recommendations
        var products = _repository.GetAllProducts();
        var recommendations = _recommendationEngine.GetRecommendations(user.UserId);
        return new ProductList(products, recommendations);
    }
    
    private ProductList GetPublicProducts(string ipAddress)
    {
        // Generic product list
        _rateLimiter.Check(ipAddress);
        return new ProductList(_repository.GetFeaturedProducts(), []);
    }
}
```

## Testing

```csharp
public class OrderServiceTests
{
    [Fact]
    public void PlaceOrder_CreatesOrderForAuthenticatedUser()
    {
        // Arrange
        var user = TestData.CreateAuthenticatedUser(
            userId: UserId.New(),
            username: "testuser",
            email: Email.Create("test@example.com").Value,
            roles: [Role.User]
        );
        
        var command = new PlaceOrderCommand
        {
            Items = [new OrderItem("Product1", 1)]
        };
        
        var service = new OrderService(_mockRepository.Object);
        
        // Act
        service.PlaceOrder(user, command);
        
        // Assert
        _mockRepository.Verify(r => r.Save(It.Is<Order>(o => o.UserId == user.UserId)));
    }
}

// Test helper
public static class TestData
{
    public static AuthenticatedUser CreateAuthenticatedUser(
        UserId userId,
        string username,
        Email email,
        Role[] roles)
    {
        // Using reflection or internal visibility to create for tests
        return new AuthenticatedUser(userId, username, email, roles.ToHashSet());
    }
}
```

## Why It's a Problem

1. **HTTP coupling**: Business logic depends on web infrastructure
2. **No compile-time guarantee**: Authentication might not have happened
3. **Raw IDs**: `Guid userId` could be anyone, not necessarily authenticated user
4. **Testing complexity**: Must mock `HttpContext` for unit tests
5. **Authorization bugs**: Easy to forget to check who the user is

## Symptoms

- `IHttpContextAccessor` injected into services
- `HttpContext.User` accessed in business logic
- Methods accepting `Guid userId` with no authorization
- Comments like `// Must be called with authenticated user`
- Unit tests that mock HTTP contexts

## Benefits

- **Compile-time enforcement**: Cannot call methods without authenticated user
- **HTTP-independent**: Business logic doesn't know about web infrastructure
- **Self-documenting**: `PlaceOrder(AuthenticatedUser user)` makes requirements clear
- **Testable**: Easy to create authenticated user instances for tests
- **Authorization built-in**: User identity is proven by the type

## See Also

- [Capability Security](./capability-security.md) — authorization tokens
- [Enforcing Call Order](./enforcing-call-order.md) — type-based sequencing
- [Honest Functions](./honest-functions.md) — explicit signatures
- [Strongly Typed IDs](./strongly-typed-ids.md) — preventing ID confusion
