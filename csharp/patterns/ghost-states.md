# Ghost States (Context-Specific Models)

> Objects with properties that are null or invalid depending on how they were retrieved—use context-specific types instead.

## Problem

A "ghost state" occurs when an object exists but portions of it are unpopulated depending on context. Common with ORMs and APIs, these objects lie about their completeness—the type says `Orders` exists, but it might be null or empty because you forgot `.Include()`.

## Example

### ❌ Before

```csharp
// The "God Class" that represents the database table
public class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    public byte[] ProfilePicture { get; set; }  // Heavy! Only loaded sometimes.
    public List<Order> Orders { get; set; }     // Expensive! Only loaded if .Include() used.
}

public class UserService
{
    public void DisplayDashboard(int userId)
    {
        User user = _repo.GetBasicUser(userId);  // Loads ID and Username only

        // BUG: NullReferenceException or empty list logic error
        // because Orders weren't loaded in this context
        Console.WriteLine($"Recent Orders: {user.Orders.Count}"); 
    }
}

// Elsewhere, a different bug...
public void ShowProfile(int userId)
{
    User user = _repo.GetUserWithOrders(userId);  // Loads orders but not profile picture
    
    // BUG: ProfilePicture is null here, but the type doesn't tell us that
    await SaveThumbnail(user.ProfilePicture);  // NullReferenceException
}
```

### ✅ After

```csharp
// Context-specific types that represent exactly what's available

// Lightweight model for lists/headers
public sealed record UserSummary(int Id, string Username);

// Profile page model
public sealed record UserProfile(int Id, string Username, byte[] ProfilePicture);

// Order processing model
public sealed record UserWithOrders(int Id, string Username, IReadOnlyList<Order> Orders);

// Full user for admin views (when you actually need everything)
public sealed record UserFull(
    int Id, 
    string Username, 
    byte[] ProfilePicture, 
    IReadOnlyList<Order> Orders);

public class UserService
{
    public void DisplayDashboard(int userId)
    {
        // Repository returns exactly what is promised
        UserWithOrders user = _repo.GetUserWithOrders(userId);

        // Safe: The type guarantees Orders exists
        Console.WriteLine($"Recent Orders: {user.Orders.Count}");
    }

    public void ShowProfile(int userId)
    {
        // This method gets a UserProfile, which guarantees ProfilePicture
        UserProfile profile = _repo.GetUserProfile(userId);
        
        // Safe: ProfilePicture is guaranteed to be present
        await SaveThumbnail(profile.ProfilePicture);
    }

    public void ListUsers()
    {
        // Lightweight query - no joins, no blobs
        IReadOnlyList<UserSummary> users = _repo.GetAllUserSummaries();
        
        // Can't accidentally access Orders here—UserSummary doesn't have them
        foreach (var user in users)
        {
            Console.WriteLine(user.Username);
        }
    }
}
```

## Why It's a Problem

1. **Lying types**: The `User` type claims it *has* Orders. But in `GetBasicUser` context, it doesn't.

2. **Temporal coupling**: Property validity depends on *when* and *how* the object was created, not the type definition.

3. **Defensive coding**: Consumers must constantly check `if (user.Orders != null)` everywhere.

4. **Hidden performance**: Returning `User` doesn't reveal whether you're fetching 3 columns or 30 with 4 joins.

## Symptoms

- Properties with comments like `// Can be null if Include() wasn't called`
- Massive classes where 50% of fields are null in any given scenario
- `NotImplementedException` or `InvalidOperationException` in property getters
- Lazy loading triggered unexpectedly, causing N+1 queries
- Different repository methods returning the same type with different "fullness"
- Runtime null checks that should be compile-time guarantees

## Repository Pattern with Context-Specific Returns

```csharp
public interface IUserRepository
{
    // Each method returns exactly what it fetches—no more, no less
    Task<UserSummary?> GetSummary(UserId id);
    Task<UserProfile?> GetProfile(UserId id);
    Task<UserWithOrders?> GetWithOrders(UserId id);
    Task<IReadOnlyList<UserSummary>> GetAllSummaries();
}

public class UserRepository : IUserRepository
{
    public async Task<UserSummary?> GetSummary(UserId id)
    {
        return await _db.Users
            .Where(u => u.Id == id.Value)
            .Select(u => new UserSummary(u.Id, u.Username))
            .FirstOrDefaultAsync();
    }

    public async Task<UserWithOrders?> GetWithOrders(UserId id)
    {
        return await _db.Users
            .Where(u => u.Id == id.Value)
            .Select(u => new UserWithOrders(
                u.Id, 
                u.Username, 
                u.Orders.ToList()))
            .FirstOrDefaultAsync();
    }
}
```

## Mapping Between Contexts

When you need to convert between context-specific types:

```csharp
public static class UserMappings
{
    // Expand: fetch additional data to create a richer type
    public static async Task<UserWithOrders> WithOrders(
        this UserSummary summary, 
        IOrderRepository orderRepo)
    {
        var orders = await orderRepo.GetByUser(new UserId(summary.Id));
        return new UserWithOrders(summary.Id, summary.Username, orders);
    }

    // Contract: create a lighter type from a heavier one (no async needed)
    public static UserSummary ToSummary(this UserWithOrders user)
        => new(user.Id, user.Username);

    public static UserSummary ToSummary(this UserProfile profile)
        => new(profile.Id, profile.Username);
}
```

## EF Core: Entity vs Read Model

Keep your EF entities internal, expose context-specific read models:

```csharp
// Internal: EF Core entity with all navigation properties
internal class UserEntity
{
    public int Id { get; set; }
    public string Username { get; set; } = "";
    public byte[]? ProfilePicture { get; set; }
    public List<OrderEntity> Orders { get; set; } = new();
}

// Public: Context-specific models that can't have ghost states
public sealed record UserSummary(int Id, string Username);
public sealed record UserProfile(int Id, string Username, byte[] ProfilePicture);
public sealed record UserWithOrders(int Id, string Username, IReadOnlyList<Order> Orders);
```

## Benefits

- **Compile-time confidence**: Can't access data that wasn't fetched
- **Performance visibility**: `UserSummary` vs `UserFull` makes cost obvious
- **No defensive null checks**: The type tells you exactly what's available
- **Self-documenting queries**: Method return types reveal the query shape
- **Easier testing**: No need to set up "partially loaded" objects

## See Also

- [Honest Functions](./honest-functions.md)
- [Primitive Obsession](./primitive-obsession.md)
- [Value Semantics](./value-semantics.md)
