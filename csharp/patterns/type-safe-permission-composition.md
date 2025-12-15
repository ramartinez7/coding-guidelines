# Type-Safe Permission Composition (Combining Capabilities)

> Permissions checked with boolean AND/OR operations allow privilege escalation—use typed permission composition to enforce least privilege through the type system.

## Problem

Permission systems using boolean flags or string-based permission names can be combined incorrectly, allowing operations that should require multiple permissions to execute with only one. The compiler can't verify that all required permissions are present.

## Example

### ❌ Before

```csharp
public class DocumentService
{
    public void DeleteDocument(User user, DocumentId id)
    {
        if (!user.HasPermission("documents.delete"))
            throw new UnauthorizedException();
        
        // Actually requires both delete AND ownership check
        _repository.Delete(id);
    }
}
```

### ✅ After

```csharp
public interface IPermission { }

public sealed record DeleteDocumentPermission : IPermission;
public sealed record ViewDocumentPermission : IPermission;
public sealed record OwnershipPermission : IPermission;

public sealed record PermissionSet<T1> where T1 : IPermission
{
    private PermissionSet() { }
    
    public static PermissionSet<T1> Create() => new();
}

public sealed record PermissionSet<T1, T2> 
    where T1 : IPermission
    where T2 : IPermission
{
    private PermissionSet() { }
    
    public static PermissionSet<T1, T2> Create() => new();
}

public static class PermissionExtensions
{
    public static PermissionSet<T1, T2> And<T1, T2>(
        this PermissionSet<T1> first,
        PermissionSet<T2> second)
        where T1 : IPermission
        where T2 : IPermission
    {
        return PermissionSet<T1, T2>.Create();
    }
}

public class DocumentService
{
    public void DeleteDocument(
        PermissionSet<DeleteDocumentPermission, OwnershipPermission> permissions,
        DocumentId id)
    {
        // Type system guarantees both permissions present
        _repository.Delete(id);
    }
}

// Usage
var deletePermission = PermissionSet<DeleteDocumentPermission>.Create();
var ownerPermission = PermissionSet<OwnershipPermission>.Create();
var combined = deletePermission.And(ownerPermission);

service.DeleteDocument(combined, documentId);
```

## See Also

- [Capability Security](./capability-security.md)
- [Authentication Context](./authentication-context.md)
- [Phantom Types](./phantom-types.md)
