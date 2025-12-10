# Algebraic Effects Emulation (Effect Handlers in C#)

> Side effects hidden in methods make code unpredictable—make effects explicit in types to enable effect handling and interpretation.

## Problem

Methods that perform I/O, logging, or other side effects hide these effects from callers, making code hard to reason about and test.

## Example

### ❌ Hidden Effects

```csharp
public class UserService
{
    public async Task<User> GetUser(int id)
    {
        // Hidden effects: database access, logging
        _logger.LogInformation($"Getting user {id}");
        var user = await _db.Users.FindAsync(id);
        _logger.LogInformation($"Found user {user.Name}");
        return user;
    }
}
```

### ✅ Explicit Effects

```csharp
// Effect as a type
public abstract record Effect<T>
{
    public sealed record Pure(T Value) : Effect<T>;
    public sealed record Log(string Message, Func<Effect<T>> Continue) : Effect<T>;
    public sealed record ReadDb<TEntity>(int Id, Func<TEntity, Effect<T>> Continue) : Effect<T>;
}

// Program that describes effects without executing them
public static Effect<User> GetUserProgram(int userId) =>
    new Effect<User>.Log(
        $"Getting user {userId}",
        () => new Effect<User>.ReadDb<User>(
            userId,
            user => new Effect<User>.Log(
                $"Found user {user.Name}",
                () => new Effect<User>.Pure(user))));

// Interpreter: decides how to handle effects
public sealed class EffectInterpreter
{
    private readonly ILogger _logger;
    private readonly DbContext _db;
    
    public async Task<T> Run<T>(Effect<T> effect)
    {
        return effect switch
        {
            Effect<T>.Pure pure => pure.Value,
            
            Effect<T>.Log log =>
            {
                _logger.LogInformation(log.Message);
                return await Run(log.Continue());
            },
            
            Effect<T>.ReadDb<TEntity> read =>
            {
                var entity = await _db.Set<TEntity>().FindAsync(read.Id);
                return await Run(read.Continue(entity));
            },
            
            _ => throw new InvalidOperationException("Unknown effect")
        };
    }
}

// Test interpreter: no I/O, just returns mock data
public sealed class TestEffectInterpreter
{
    private readonly Dictionary<int, User> _mockUsers = new();
    private readonly List<string> _logs = new();
    
    public T Run<T>(Effect<T> effect)
    {
        return effect switch
        {
            Effect<T>.Pure pure => pure.Value,
            
            Effect<T>.Log log =>
            {
                _logs.Add(log.Message);
                return Run(log.Continue());
            },
            
            Effect<T>.ReadDb<User> read =>
            {
                var user = _mockUsers[read.Id];
                return Run(read.Continue(user));
            },
            
            _ => throw new InvalidOperationException("Unknown effect")
        };
    }
}
```

## See Also

- [Honest Functions](./honest-functions.md)
- [Type-Safe Functional Core](./type-safe-functional-core.md)
