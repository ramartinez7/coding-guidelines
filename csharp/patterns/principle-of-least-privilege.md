# Principle of Least Privilege (Type-Safe Privilege Boundaries)

> Granting broad permissions and trusting code to self-limit—use types to enforce that code can only access the minimum privileges required.

## Problem

Traditional security grants permissions broadly (admin vs. user) and trusts code to check before performing sensitive operations. This creates opportunities for privilege escalation, accidental misuse, and security holes when developers forget to check permissions.

## Example

### ❌ Before

```csharp
public class DocumentService
{
    IDocumentRepository _repository;
    IAuthorizationService _authService;
    
    // Service has access to ALL documents—trust it to check permissions
    public async Task<Document> GetDocument(UserId userId, DocumentId documentId)
    {
        // Easy to forget this check
        if (!await _authService.CanView(userId, documentId))
            throw new UnauthorizedException();
        
        return await _repository.GetById(documentId);
    }
    
    public async Task DeleteDocument(UserId userId, DocumentId documentId)
    {
        // Different permissions required—must remember to check
        if (!await _authService.CanDelete(userId, documentId))
            throw new UnauthorizedException();
        
        await _repository.Delete(documentId);
    }
    
    // Oops! Forgot to check permissions
    public async Task UpdateDocument(UserId userId, DocumentId documentId, string content)
    {
        var doc = await _repository.GetById(documentId);
        doc.Content = content;
        await _repository.Update(doc);  // 💥 Security hole
    }
}

// Repository has god-mode access to all documents
public interface IDocumentRepository
{
    Task<Document> GetById(DocumentId id);
    Task Update(Document doc);
    Task Delete(DocumentId id);
    // Any code with this interface can do anything!
}
```

**Problems:**
- Service has access to all operations, trusts itself to check
- Easy to forget authorization checks
- New methods might skip checks
- Repository provides unrestricted access
- Code review is the only safety net

### ✅ After (Type-Safe Privilege Boundaries)

```csharp
/// <summary>
/// Repository scoped to documents a specific user can view.
/// Cannot access documents outside this scope.
/// </summary>
public interface IViewableDocumentRepository
{
    Task<Option<Document>> GetById(DocumentId id);
    Task<IReadOnlyList<Document>> GetAll();
}

/// <summary>
/// Repository scoped to documents a specific user can edit.
/// Cannot perform operations outside this scope.
/// </summary>
public interface IEditableDocumentRepository : IViewableDocumentRepository
{
    Task Update(Document document);
}

/// <summary>
/// Repository scoped to documents a specific user can delete.
/// </summary>
public interface IDeletableDocumentRepository : IViewableDocumentRepository
{
    Task Delete(DocumentId id);
}

/// <summary>
/// Factory that creates privilege-scoped repositories.
/// Only factory can create them—ensures authorization checked.
/// </summary>
public class DocumentRepositoryFactory
{
    readonly IAuthorizationService _authService;
    readonly IDocumentRepository _underlyingRepo;
    
    public DocumentRepositoryFactory(
        IAuthorizationService authService,
        IDocumentRepository underlyingRepo)
    {
        this._authService = authService;
        this._underlyingRepo = underlyingRepo;
    }
    
    /// <summary>
    /// Creates repository scoped to documents user can view.
    /// Authorization checked HERE, once.
    /// </summary>
    public async Task<IViewableDocumentRepository> CreateViewableScope(UserId userId)
    {
        var viewableDocIds = await _authService.GetViewableDocuments(userId);
        return new ViewableDocumentRepository(_underlyingRepo, viewableDocIds);
    }
    
    /// <summary>
    /// Creates repository scoped to documents user can edit.
    /// </summary>
    public async Task<IEditableDocumentRepository> CreateEditableScope(UserId userId)
    {
        var editableDocIds = await _authService.GetEditableDocuments(userId);
        return new EditableDocumentRepository(_underlyingRepo, editableDocIds);
    }
    
    /// <summary>
    /// Creates repository scoped to documents user can delete.
    /// </summary>
    public async Task<IDeletableDocumentRepository> CreateDeletableScope(UserId userId)
    {
        var deletableDocIds = await _authService.GetDeletableDocuments(userId);
        return new DeletableDocumentRepository(_underlyingRepo, deletableDocIds);
    }
}

/// <summary>
/// Implementation that enforces view-only access.
/// </summary>
sealed class ViewableDocumentRepository : IViewableDocumentRepository
{
    readonly IDocumentRepository _underlying;
    readonly IReadOnlySet<DocumentId> _allowedIds;
    
    internal ViewableDocumentRepository(
        IDocumentRepository underlying,
        IReadOnlySet<DocumentId> allowedIds)
    {
        _underlying = underlying;
        _allowedIds = allowedIds;
    }
    
    public async Task<Option<Document>> GetById(DocumentId id)
    {
        // Enforce scope—can only access allowed documents
        if (!_allowedIds.Contains(id))
            return Option<Document>.None;
        
        return await _underlying.GetById(id);
    }
    
    public async Task<IReadOnlyList<Document>> GetAll()
    {
        var all = await _underlying.GetAll();
        return all.Where(d => _allowedIds.Contains(d.Id)).ToList();
    }
}

/// <summary>
/// Implementation that enforces edit access.
/// </summary>
sealed class EditableDocumentRepository : IEditableDocumentRepository
{
    readonly IDocumentRepository _underlying;
    readonly IReadOnlySet<DocumentId> _allowedIds;
    
    internal EditableDocumentRepository(
        IDocumentRepository underlying,
        IReadOnlySet<DocumentId> allowedIds)
    {
        _underlying = underlying;
        _allowedIds = allowedIds;
    }
    
    public async Task<Option<Document>> GetById(DocumentId id)
    {
        if (!_allowedIds.Contains(id))
            return Option<Document>.None;
        
        return await _underlying.GetById(id);
    }
    
    public async Task<IReadOnlyList<Document>> GetAll()
    {
        var all = await _underlying.GetAll();
        return all.Where(d => _allowedIds.Contains(d.Id)).ToList();
    }
    
    public async Task Update(Document document)
    {
        // Enforce scope—can only update allowed documents
        if (!_allowedIds.Contains(document.Id))
            throw new UnauthorizedException(
                $"No edit permission for document {document.Id}");
        
        await _underlying.Update(document);
    }
}

// Service receives scoped repository—no need to check permissions
public class DocumentService
{
    // Method signature proves authorization checked
    public async Task<Document> GetDocument(
        IViewableDocumentRepository repository,
        DocumentId documentId)
    {
        // No authorization check needed—repository is already scoped
        return await repository.GetById(documentId)
            .Match(
                onSome: doc => doc,
                onNone: () => throw new NotFoundException());
    }
    
    // Cannot be called without edit permission—enforced by type
    public async Task UpdateDocument(
        IEditableDocumentRepository repository,
        DocumentId documentId,
        string content)
    {
        // No authorization check needed—repository enforces scope
        var docResult = await repository.GetById(documentId);
        
        if (!docResult.HasValue)
            throw new NotFoundException();
        
        var doc = docResult.Value with { Content = content };
        await repository.Update(doc);  // Safe—repository checks permission
    }
    
    // Cannot be called without delete permission—enforced by type
    public async Task DeleteDocument(
        IDeletableDocumentRepository repository,
        DocumentId documentId)
    {
        // No authorization check needed
        await repository.Delete(documentId);
    }
}

// Controller creates scoped repository
public class DocumentController : ControllerBase
{
    DocumentRepositoryFactory _repoFactory;
    DocumentService _service;
    
    [HttpPut("documents/{documentId}")]
    public async Task<IActionResult> UpdateDocument(Guid documentId, [FromBody] UpdateDocumentRequest request)
    {
        var userId = GetCurrentUserId();
        var docId = new DocumentId(documentId);
        
        // Create scoped repository—authorization happens HERE
        var editableRepo = await _repoFactory.CreateEditableScope(userId);
        
        // Service can only do what repository allows
        await _service.UpdateDocument(editableRepo, docId, request.Content);
        
        return Ok();
    }
}
```

## File System Access with Least Privilege

```csharp
/// <summary>
/// Read-only file system access.
/// </summary>
public interface IReadableFileSystem
{
    Task<string> ReadFile(FilePath path);
    Task<bool> FileExists(FilePath path);
    Task<IReadOnlyList<FilePath>> ListFiles(DirectoryPath directory);
}

/// <summary>
/// Write access to specific directory only.
/// </summary>
public interface IWritableFileSystem : IReadableFileSystem
{
    Task WriteFile(FilePath path, string content);
    Task DeleteFile(FilePath path);
}

/// <summary>
/// Factory creates scoped file system access.
/// </summary>
public class FileSystemFactory
{
    /// <summary>
    /// Creates read-only access to specific directory.
    /// Cannot escape directory—enforced by FilePath validation.
    /// </summary>
    public IReadableFileSystem CreateReadableScope(DirectoryPath allowedDirectory)
    {
        return new ReadableFileSystem(allowedDirectory);
    }
    
    /// <summary>
    /// Creates write access to specific directory.
    /// Requires explicit authorization.
    /// </summary>
    public IWritableFileSystem CreateWritableScope(
        DirectoryPath allowedDirectory,
        Capability<WriteToDirectory> capability)
    {
        return new WritableFileSystem(allowedDirectory);
    }
}

sealed class ReadableFileSystem : IReadableFileSystem
{
    readonly DirectoryPath _allowedDirectory;
    
    internal ReadableFileSystem(DirectoryPath allowedDirectory)
    {
        this._allowedDirectory = allowedDirectory;
    }
    
    public async Task<string> ReadFile(FilePath path)
    {
        // Validate path is within allowed directory
        if (!path.IsUnder(_allowedDirectory))
            throw new UnauthorizedException($"Path {path} not in allowed directory");
        
        return await File.ReadAllTextAsync(path.ToString());
    }
    
    public async Task<bool> FileExists(FilePath path)
    {
        if (!path.IsUnder(_allowedDirectory))
            return false;
        
        return File.Exists(path.ToString());
    }
    
    public async Task<IReadOnlyList<FilePath>> ListFiles(DirectoryPath directory)
    {
        if (!directory.IsUnder(_allowedDirectory))
            throw new UnauthorizedException($"Directory {directory} not in allowed directory");
        
        var files = Directory.GetFiles(directory.ToString());
        return files.Select(f => FilePath.From(f)).ToList();
    }
}
```

## Database Access with Least Privilege

```csharp
/// <summary>
/// Read-only database access for specific user's data.
/// </summary>
public interface IUserDataReader
{
    Task<Option<User>> GetCurrentUser();
    Task<IReadOnlyList<Order>> GetUserOrders();
    Task<IReadOnlyList<Payment>> GetUserPayments();
}

/// <summary>
/// Read-write access to user's own data.
/// Cannot modify other users' data.
/// </summary>
public interface IUserDataWriter : IUserDataReader
{
    Task UpdateProfile(UserProfile profile);
    Task AddPaymentMethod(PaymentMethod method);
}

/// <summary>
/// Admin access to all user data.
/// Requires admin capability.
/// </summary>
public interface IAdminUserRepository
{
    Task<Option<User>> GetUserById(UserId id);
    Task<IReadOnlyList<User>> GetAllUsers();
    Task DisableUser(UserId id, string reason);
}

public class UserRepositoryFactory
{
    readonly IUserRepository _underlying;
    readonly IAuthorizationService _authService;
    
    /// <summary>
    /// Creates repository scoped to current user's data.
    /// User can only access their own data.
    /// </summary>
    public async Task<IUserDataReader> CreateUserScope(UserId userId)
    {
        return new UserDataReader(_underlying, userId);
    }
    
    /// <summary>
    /// Creates read-write scope for user's own data.
    /// </summary>
    public async Task<IUserDataWriter> CreateUserWriteScope(UserId userId)
    {
        return new UserDataWriter(_underlying, userId);
    }
    
    /// <summary>
    /// Creates admin scope—requires admin capability.
    /// </summary>
    public IAdminUserRepository CreateAdminScope(Capability<AdminAccess> capability)
    {
        return new AdminUserRepository(_underlying, capability.AuthorizedUser);
    }
}

sealed class UserDataReader : IUserDataReader
{
    readonly IUserRepository _underlying;
    readonly UserId _scopedUserId;
    
    internal UserDataReader(IUserRepository underlying, UserId scopedUserId)
    {
        this._underlying = underlying;
        this._scopedUserId = scopedUserId;
    }
    
    public async Task<Option<User>> GetCurrentUser()
    {
        // Can only access own data
        return await _underlying.GetById(_scopedUserId);
    }
    
    public async Task<IReadOnlyList<Order>> GetUserOrders()
    {
        // Can only access own orders
        return await _underlying.GetOrdersByUserId(_scopedUserId);
    }
    
    public async Task<IReadOnlyList<Payment>> GetUserPayments()
    {
        // Can only access own payments
        return await _underlying.GetPaymentsByUserId(_scopedUserId);
    }
}
```

## API Client with Least Privilege

```csharp
/// <summary>
/// API client with read-only access.
/// </summary>
public interface IReadOnlyApiClient
{
    Task<Option<Product>> GetProduct(ProductId id);
    Task<IReadOnlyList<Product>> SearchProducts(string query);
}

/// <summary>
/// API client that can create products.
/// Requires write capability.
/// </summary>
public interface IProductCreator : IReadOnlyApiClient
{
    Task<Result<ProductId, Error>> CreateProduct(CreateProductRequest request);
}

/// <summary>
/// API client that can update products.
/// Requires update capability.
/// </summary>
public interface IProductUpdater : IReadOnlyApiClient
{
    Task<Result<Unit, Error>> UpdateProduct(ProductId id, UpdateProductRequest request);
}

/// <summary>
/// API client that can delete products.
/// Requires delete capability—most privileged.
/// </summary>
public interface IProductDeleter : IReadOnlyApiClient
{
    Task<Result<Unit, Error>> DeleteProduct(ProductId id);
}

public class ApiClientFactory
{
    readonly HttpClient _httpClient;
    
    public IReadOnlyApiClient CreateReadOnlyClient(ApiKey readKey)
    {
        return new ReadOnlyApiClient(_httpClient, readKey);
    }
    
    public IProductCreator CreateProductCreator(ApiKey writeKey)
    {
        return new ProductCreatorClient(_httpClient, writeKey);
    }
    
    public IProductUpdater CreateProductUpdater(ApiKey updateKey)
    {
        return new ProductUpdaterClient(_httpClient, updateKey);
    }
    
    public IProductDeleter CreateProductDeleter(ApiKey deleteKey)
    {
        // Most privileged—requires strongest key
        return new ProductDeleterClient(_httpClient, deleteKey);
    }
}
```

## Key Principles

### 1. Narrow Interfaces

Create interfaces that expose only the operations needed:

```csharp
// ❌ Too broad
public interface IRepository<T>
{
    T Get(int id);
    void Add(T entity);
    void Update(T entity);
    void Delete(int id);
}

// ✅ Narrow, composable
public interface IReadRepository<T>
{
    T Get(int id);
}

public interface IWriteRepository<T> : IReadRepository<T>
{
    void Add(T entity);
    void Update(T entity);
}
```

### 2. Capability Requirements

Require capability tokens for privileged operations:

```csharp
// Method signature proves authorization
public void DeleteUser(
    Capability<DeleteUser> capability,
    UserId userId)
{
    // No check needed—capability proves permission
}
```

### 3. Scoped Repositories

Create repositories scoped to allowed resources:

```csharp
// Repository only has access to allowed IDs
sealed class ScopedRepository
{
    readonly IReadOnlySet<TId> _allowedIds;
    
    // Cannot access IDs outside this set
}
```

### 4. Factory Pattern

Use factories to enforce authorization at creation:

```csharp
public class RepositoryFactory
{
    // Authorization checked HERE
    public IScopedRepository Create(UserId userId)
    {
        var allowed = _authService.GetAllowedResources(userId);
        return new ScopedRepository(allowed);
    }
}
```

## Why It's a Problem

1. **Broad permissions**: Code has access to more than it needs
2. **Trust-based security**: Relies on remembering to check
3. **Privilege escalation**: Easy to accidentally grant too much access
4. **Hidden vulnerabilities**: Forgotten checks create security holes
5. **Audit difficulty**: Hard to trace who can do what

## Symptoms

- Global admin flags or connection strings
- Services with access to all data
- Methods that check permissions internally
- Comments like "must be admin to call this"
- Security bugs from forgotten authorization checks

## Benefits

- **Compile-time enforcement**: Cannot call without privilege
- **Principle of least privilege**: Only grant minimum access needed
- **Explicit contracts**: Method signature shows requirements
- **Audit trail**: Clear who has access to what
- **Defense in depth**: Multiple layers enforce scope

## Trade-offs

- **More interfaces**: Each privilege level needs interface
- **Factory complexity**: Must create scoped instances
- **Pass-through parameters**: Must thread scoped repositories
- **Learning curve**: Team must understand privilege boundaries

## See Also

- [Capability Security](./capability-security.md) — unforgeable authorization tokens
- [Authentication Context](./authentication-context.md) — type-safe identity
- [Assembly Isolation](./assembly-isolation.md) — enforcing boundaries
- [Type Witnesses](./type-witnesses.md) — compile-time proofs
