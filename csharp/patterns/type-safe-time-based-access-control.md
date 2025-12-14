# Type-Safe Time-Based Access Control

> Access control based on time windows checked at runtime—use typed temporal permissions to enforce time-based access at compile time.

## Problem

Time-based access control (e.g., business hours only, temporary access) implemented with runtime date checks allows access outside valid windows when checks are forgotten or implemented incorrectly. The compiler can't verify that temporal constraints are enforced.

## Example

### ❌ Before

```csharp
public class DocumentService
{
    public Document GetDocument(User user, DocumentId id)
    {
        // Runtime check—easy to forget
        if (DateTime.Now.Hour < 9 || DateTime.Now.Hour >= 17)
            throw new UnauthorizedException("Access only during business hours");
        
        return _repository.GetById(id);
    }
    
    public void GrantTemporaryAccess(User user, DateTime expiresAt)
    {
        // Runtime expiration check scattered throughout code
        _tempAccess[user.Id] = expiresAt;
    }
}
```

### ✅ After

```csharp
public interface ITemporalConstraint
{
    bool IsValid(DateTimeOffset at);
}

public sealed record BusinessHours : ITemporalConstraint
{
    public bool IsValid(DateTimeOffset at)
    {
        var hour = at.Hour;
        return hour >= 9 && hour < 17 && at.DayOfWeek != DayOfWeek.Saturday 
            && at.DayOfWeek != DayOfWeek.Sunday;
    }
}

public sealed record ExpiringAccess(DateTimeOffset ExpiresAt) : ITemporalConstraint
{
    public bool IsValid(DateTimeOffset at) => at < ExpiresAt;
}

public sealed record TemporalPermission<TPermission, TConstraint>
    where TPermission : IPermission
    where TConstraint : ITemporalConstraint
{
    public TPermission Permission { get; }
    public TConstraint Constraint { get; }
    public DateTimeOffset GrantedAt { get; }
    
    private TemporalPermission(
        TPermission permission,
        TConstraint constraint,
        DateTimeOffset grantedAt)
    {
        Permission = permission;
        Constraint = constraint;
        GrantedAt = grantedAt;
    }
    
    public static Result<TemporalPermission<TPermission, TConstraint>, string> Grant(
        TPermission permission,
        TConstraint constraint)
    {
        var now = DateTimeOffset.UtcNow;
        
        if (!constraint.IsValid(now))
            return Result<TemporalPermission<TPermission, TConstraint>, string>
                .Failure("Constraint not valid at grant time");
        
        return Result<TemporalPermission<TPermission, TConstraint>, string>
            .Success(new TemporalPermission<TPermission, TConstraint>(
                permission,
                constraint,
                now));
    }
    
    public bool IsCurrentlyValid() => Constraint.IsValid(DateTimeOffset.UtcNow);
}

public sealed record ViewDocumentPermission : IPermission;

// Usage: Type-safe temporal permissions
public class DocumentService
{
    public Result<Document, string> GetDocument(
        TemporalPermission<ViewDocumentPermission, BusinessHours> permission,
        DocumentId id)
    {
        // Type system enforces permission exists
        // Still need runtime check for temporal validity
        if (!permission.IsCurrentlyValid())
            return Result<Document, string>.Failure("Access outside business hours");
        
        var doc = _repository.GetById(id);
        return Result<Document, string>.Success(doc);
    }
    
    public Result<TemporalPermission<ViewDocumentPermission, ExpiringAccess>, string> 
        GrantTemporaryAccess(User user, TimeSpan duration)
    {
        var expiresAt = DateTimeOffset.UtcNow.Add(duration);
        var constraint = new ExpiringAccess(expiresAt);
        
        return TemporalPermission<ViewDocumentPermission, ExpiringAccess>
            .Grant(new ViewDocumentPermission(), constraint);
    }
}
```

## See Also

- [Capability Security](./capability-security.md)
- [Type-Safe Permission Composition](./type-safe-permission-composition.md)
- [Temporal Safety](./temporal-safety.md)
