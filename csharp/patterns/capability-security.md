# Capability Security (Token Types)

> Checking authorization in one place and assuming it was checked in the sensitive method—use capability tokens to make authorization proof part of the type system.

## Problem

In many systems, security checks are decoupled from actions. You call `IsAdmin()` somewhere, then assume the caller "must have checked" before calling the sensitive method. This assumption is invisible to the compiler and easy to violate during refactoring.

## Example

### ❌ Before

```csharp
public class DocumentService
{
    public void DeleteDocument(UserId userId, DocumentId documentId)
    {
        // Did the caller check authorization? We have to trust them...
        var document = _repository.Get(documentId);
        _repository.Delete(document);
        _auditLog.Record($"Document {documentId} deleted by {userId}");
    }
}

public class DocumentController
{
    public IActionResult Delete(Guid documentId)
    {
        var userId = GetCurrentUser();
        
        // Easy to forget this check
        if (!_authService.CanDeleteDocuments(userId))
            return Forbid();
        
        _documentService.DeleteDocument(userId, documentId);
        return Ok();
    }
}

// Another developer adds a new endpoint...
public class AdminController
{
    public IActionResult BulkDelete(Guid[] documentIds)
    {
        var userId = GetCurrentUser();
        
        // Oops! Forgot to check authorization
        foreach (var id in documentIds)
            _documentService.DeleteDocument(userId, id);  // 💥 Security hole
        
        return Ok();
    }
}
```

### ✅ After

```csharp
/// <summary>
/// Marker interface for all actions that require authorization.
/// </summary>
public interface IRequiresAuthorization { }

/// <summary>
/// Proof that the holder has been authorized to perform action T.
/// Cannot be constructed outside the authorization service.
/// </summary>
public sealed class Capability<TAction> where TAction : IRequiresAuthorization
{
    public UserId AuthorizedUser { get; }
    public DateTime AuthorizedAt { get; }
    
    internal Capability(UserId user)
    {
        AuthorizedUser = user;
        AuthorizedAt = DateTime.UtcNow;
    }
}

// Define actions as marker types
public sealed record DeleteDocument(DocumentId Id) : IRequiresAuthorization;
public sealed record EditDocument(DocumentId Id) : IRequiresAuthorization;
public sealed record CreateDocument : IRequiresAuthorization;
public sealed record BulkDeleteDocuments(IReadOnlyList<DocumentId> Ids) : IRequiresAuthorization;

public class AuthorizationService
{
    // Single generic method handles all actions
    public Option<Capability<TAction>> Authorize<TAction>(UserId userId, TAction action)
        where TAction : IRequiresAuthorization
    {
        if (!IsAuthorized(userId, action))
            return Option<Capability<TAction>>.None;
        
        return Option<Capability<TAction>>.Some(new Capability<TAction>(userId));
    }
    
    private bool IsAuthorized<TAction>(UserId userId, TAction action)
        where TAction : IRequiresAuthorization
    {
        // Pattern match on action type to determine authorization rules
        return action switch
        {
            DeleteDocument delete => CanDeleteDocument(userId, delete.Id),
            EditDocument edit => CanEditDocument(userId, edit.Id),
            CreateDocument => HasRole(userId, Role.Editor),
            BulkDeleteDocuments => HasRole(userId, Role.Admin),
            _ => false
        };
    }
    
    private bool CanDeleteDocument(UserId userId, DocumentId docId) => /* ... */;
    private bool CanEditDocument(UserId userId, DocumentId docId) => /* ... */;
    private bool HasRole(UserId userId, Role role) => /* ... */;
}

public class DocumentService
{
    // Method requires capability for the specific action type
    public void DeleteDocument(Capability<DeleteDocument> capability, DeleteDocument action)
    {
        var document = _repository.Get(action.Id);
        _repository.Delete(document);
        _auditLog.Record($"Document {action.Id} deleted by {capability.AuthorizedUser}");
    }
    
    public void BulkDelete(Capability<BulkDeleteDocuments> capability, BulkDeleteDocuments action)
    {
        foreach (var id in action.Ids)
        {
            var document = _repository.Get(id);
            _repository.Delete(document);
        }
        _auditLog.Record($"{action.Ids.Count} documents deleted by {capability.AuthorizedUser}");
    }
}

public class DocumentController
{
    public IActionResult Delete(Guid documentId)
    {
        var userId = GetCurrentUser();
        var action = new DeleteDocument(new DocumentId(documentId));
        
        return _authService.Authorize(userId, action).Match(
            onSome: capability =>
            {
                _documentService.DeleteDocument(capability, action);
                return Ok();
            },
            onNone: () => Forbid()
        );
    }
}

public class AdminController
{
    public IActionResult BulkDelete(Guid[] documentIds)
    {
        var userId = GetCurrentUser();
        var action = new BulkDeleteDocuments(documentIds.Select(id => new DocumentId(id)).ToList());
        
        return _authService.Authorize(userId, action).Match(
            onSome: capability =>
            {
                _documentService.BulkDelete(capability, action);
                return Ok();
            },
            onNone: () => Forbid()
        );
    }
}

// Type safety still works:
var deleteAction = new DeleteDocument(docId);
var deleteCapability = _authService.Authorize(userId, deleteAction);

var bulkAction = new BulkDeleteDocuments(ids);

// ❌ Compile error: cannot convert Capability<DeleteDocument> to Capability<BulkDeleteDocuments>
_documentService.BulkDelete(deleteCapability, bulkAction);
```

## Key Design Principles

### 0. Enable Nullable Reference Types (Required)

This pattern **requires** nullable reference types to be enabled project-wide with warnings as errors. Otherwise, callers can pass `null` and bypass the capability check entirely.

```xml
<!-- In your .csproj -->
<PropertyGroup>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <!-- Or at minimum: -->
    <WarningsAsErrors>CS8600;CS8602;CS8603;CS8604;CS8625</WarningsAsErrors>
</PropertyGroup>
```

With this enabled, passing `null` to a non-nullable parameter is a compile error:

```csharp
public void Delete(Capability<DeleteDocument> capability, DeleteDocument action)
{
    // capability is guaranteed non-null by the compiler
}

// ❌ Compile error CS8625: Cannot convert null to non-nullable reference type
_documentService.Delete(null!, action);
```

> **Note**: Using a `struct` does not solve this problem—developers can still call `default(Capability<T>)` to create an invalid capability. There's no compile-time defense against `default(T)` for structs.

### 1. Generic Capability with Action Types

Instead of a separate capability class per action, use a generic `Capability<TAction>` parameterized by the action type:

```csharp
public sealed class Capability<TAction> where TAction : IRequiresAuthorization
{
    public UserId AuthorizedUser { get; }
    internal Capability(UserId user) => AuthorizedUser = user;
}
```

Actions are defined as marker types that carry their parameters:

```csharp
public sealed record DeleteDocument(DocumentId Id) : IRequiresAuthorization;
public sealed record TransferFunds(AccountId From, AccountId To, Money Amount) : IRequiresAuthorization;
```

The generic constraint ensures type safety: `Capability<DeleteDocument>` cannot be used where `Capability<TransferFunds>` is required.

### 2. Single Authorization Method

One generic method handles all action types:

```csharp
public Option<Capability<TAction>> Authorize<TAction>(UserId userId, TAction action)
    where TAction : IRequiresAuthorization
{
    if (!IsAuthorized(userId, action))
        return Option<Capability<TAction>>.None;
    
    return Option<Capability<TAction>>.Some(new Capability<TAction>(userId));
}
```

Authorization rules are centralized in `IsAuthorized`, using pattern matching:

```csharp
private bool IsAuthorized<TAction>(UserId userId, TAction action)
    where TAction : IRequiresAuthorization
{
    return action switch
    {
        DeleteDocument d => OwnsDocument(userId, d.Id) || IsAdmin(userId),
        TransferFunds t => OwnsAccount(userId, t.From) && t.Amount.Value <= GetLimit(userId),
        CreateDocument => IsEditor(userId),
        _ => false  // Deny unknown actions by default
    };
}
```

### 3. Action Carries Context

The action record itself carries all the context needed for authorization and execution:

```csharp
public sealed record TransferFunds(
    AccountId From,
    AccountId To,
    Money Amount
) : IRequiresAuthorization;

// The action has everything needed
public void Execute(Capability<TransferFunds> capability, TransferFunds action)
{
    _accountService.Transfer(action.From, action.To, action.Amount);
    _auditLog.Record($"Transfer by {capability.AuthorizedUser}: {action}");
}
```

### 4. Time-Limited Capabilities

Add expiration to the base capability if needed:

```csharp
public sealed class Capability<TAction> where TAction : IRequiresAuthorization
{
    public UserId AuthorizedUser { get; }
    public DateTime AuthorizedAt { get; }
    public DateTime? ExpiresAt { get; }
    
    public bool IsExpired => ExpiresAt.HasValue && DateTime.UtcNow > ExpiresAt.Value;
    
    internal Capability(UserId user, TimeSpan? validFor = null)
    {
        AuthorizedUser = user;
        AuthorizedAt = DateTime.UtcNow;
        ExpiresAt = validFor.HasValue ? AuthorizedAt + validFor.Value : null;
    }
}

public void Execute<TAction>(Capability<TAction> capability, TAction action)
    where TAction : IRequiresAuthorization
{
    if (capability.IsExpired)
        throw new SecurityException("Capability has expired");
    // ...
}
```

## Advanced Patterns

### Hierarchical Actions

Use inheritance for action hierarchies:

```csharp
// Base action for all document operations
public abstract record DocumentAction(DocumentId Id) : IRequiresAuthorization;

public sealed record ViewDocument(DocumentId Id) : DocumentAction(Id);
public sealed record EditDocument(DocumentId Id) : DocumentAction(Id);
public sealed record DeleteDocument(DocumentId Id) : DocumentAction(Id);

// Authorization can match on the base type
private bool IsAuthorized<TAction>(UserId userId, TAction action) => action switch
{
    DeleteDocument d => IsAdmin(userId),
    EditDocument e => OwnsDocument(userId, e.Id) || IsEditor(userId),
    ViewDocument v => true,  // Everyone can view
    DocumentAction d => false,  // Deny unknown document actions
    _ => false
};
```

### Composite Actions

```csharp
public sealed record BatchOperation<TAction>(IReadOnlyList<TAction> Actions) 
    : IRequiresAuthorization 
    where TAction : IRequiresAuthorization;

// Authorization checks all sub-actions
private bool IsAuthorized<TAction>(UserId userId, TAction action) => action switch
{
    BatchOperation<TInner> batch => batch.Actions.All(a => IsAuthorized(userId, a)),
    // ... other cases
};
```

### Role-Gated Actions

Some actions require specific roles regardless of resource ownership:

```csharp
// Marker interfaces for role requirements
public interface IAdminOnly : IRequiresAuthorization { }
public interface IModeratorOnly : IRequiresAuthorization { }

public sealed record PurgeAllData : IAdminOnly;
public sealed record BanUser(UserId Target) : IModeratorOnly;

private bool IsAuthorized<TAction>(UserId userId, TAction action) => action switch
{
    IAdminOnly => HasRole(userId, Role.Admin),
    IModeratorOnly => HasRole(userId, Role.Moderator) || HasRole(userId, Role.Admin),
    // ... other cases
};
```

## Why It's a Problem

- **Silent security holes**: Missing authorization checks aren't caught until penetration testing (or production)
- **Trust assumptions**: Methods assume "someone must have checked" without proof
- **Refactoring danger**: Moving code can accidentally bypass authorization
- **Audit difficulty**: Hard to trace which code path authorized an action
- **Testing gaps**: Unit tests often skip authorization, missing integration issues

## Symptoms

- `if (IsAdmin())` scattered throughout the codebase
- Security bugs found during code review or penetration testing
- Methods that "should only be called after authorization" (per comments)
- Duplicate authorization checks (defensive, "just in case")
- Audit logs that can't trace authorization decisions

## Benefits

- **Compile-time enforcement**: Missing authorization = compile error
- **Unforgeable proof**: Capabilities can only come from authorization service
- **Self-documenting**: Method signature shows exactly what authorization is required
- **Auditable**: Capability carries who authorized what and when
- **Testable**: Mock the authorization service, not the security checks

## Trade-offs

- **Requires nullable reference types**: Without `<Nullable>enable</Nullable>` and warnings-as-errors, callers can pass `null`
- **More types**: Each capability needs its own type
- **Explicit passing**: Capabilities must be threaded through call chains
- **Complexity**: Simple apps may not need this level of security

## See Also

- [Enforcing Call Order](./enforcing-call-order.md) — similar pattern for workflow steps
- [Type-Safe Step Builder](./type-safe-builder.md) — requiring tokens before actions
- [Honest Functions](./honest-functions.md) — making requirements explicit in signatures
