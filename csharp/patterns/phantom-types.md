# Phantom Types (Compile-Time State Tracking)

> Using runtime checks or comments to track object state—use phantom type parameters to encode state in the type system.

## Problem

Objects often go through lifecycle states (uninitialized → validated → processed → saved). Traditional approaches use enums, booleans, or runtime checks to track state. This allows invalid state transitions and provides no compile-time guarantees.

## Example

### ❌ Before

```csharp
public class Order
{
    public OrderId Id { get; set; }
    public List<OrderItem> Items { get; set; } = new();
    public OrderState State { get; set; } = OrderState.Draft;
    
    public void Validate()
    {
        if (State != OrderState.Draft)
            throw new InvalidOperationException("Can only validate draft orders");
        
        // Validation logic
        State = OrderState.Validated;
    }
    
    public void Process()
    {
        if (State != OrderState.Validated)
            throw new InvalidOperationException("Can only process validated orders");
        
        // Processing logic
        State = OrderState.Processed;
    }
    
    public void Ship()
    {
        if (State != OrderState.Processed)
            throw new InvalidOperationException("Can only ship processed orders");
        
        // Shipping logic
        State = OrderState.Shipped;
    }
}

public enum OrderState
{
    Draft,
    Validated,
    Processed,
    Shipped
}

// Problems: Can call methods in wrong order, runtime checks required
var order = new Order();
order.Ship();  // Compiles but fails at runtime
```

**Problems:**
- Runtime state checks
- Can call methods in wrong order (compiles but fails at runtime)
- State enum can be set directly, bypassing validation
- No compile-time guarantees about valid transitions

### ✅ After

```csharp
/// <summary>
/// Marker interfaces for order states (phantom types).
/// </summary>
public interface IOrderState { }
public interface IDraft : IOrderState { }
public interface IValidated : IOrderState { }
public interface IProcessed : IOrderState { }
public interface IShipped : IOrderState { }

/// <summary>
/// Order parameterized by its current state.
/// The type parameter exists only at compile time—no runtime overhead.
/// </summary>
public sealed class Order<TState> where TState : IOrderState
{
    public OrderId Id { get; }
    public IReadOnlyList<OrderItem> Items { get; }
    
    // Private constructor—can only be created through factory methods
    private Order(OrderId id, IReadOnlyList<OrderItem> items)
    {
        Id = id;
        Items = items;
    }
    
    // Factory method creates draft order
    public static Order<IDraft> CreateDraft(OrderId id, List<OrderItem> items)
    {
        return new Order<IDraft>(id, items);
    }
    
    // State transitions are methods that return new type
    public Order<IValidated> Validate(this Order<IDraft> order)
    {
        // Validation logic
        if (order.Items.Count == 0)
            throw new InvalidOperationException("Order must have items");
        
        return new Order<IValidated>(order.Id, order.Items);
    }
    
    public Order<IProcessed> Process(this Order<IValidated> order)
    {
        // Processing logic
        return new Order<IProcessed>(order.Id, order.Items);
    }
    
    public Order<IShipped> Ship(this Order<IProcessed> order)
    {
        // Shipping logic
        return new Order<IShipped>(order.Id, order.Items);
    }
}

// Extension methods for state transitions
public static class OrderExtensions
{
    public static Order<IValidated> Validate(this Order<IDraft> order)
    {
        if (order.Items.Count == 0)
            throw new InvalidOperationException("Order must have items");
        
        return Order<IValidated>.From(order);
    }
    
    public static Order<IProcessed> Process(this Order<IValidated> order)
    {
        // Processing logic
        return Order<IProcessed>.From(order);
    }
    
    public static Order<IShipped> Ship(this Order<IProcessed> order)
    {
        // Shipping logic
        return Order<IShipped>.From(order);
    }
}

// Internal helper to convert between states
internal static class Order
{
    internal static Order<TState> From<TState>(Order<IOrderState> source) 
        where TState : IOrderState
    {
        return new Order<TState>(source.Id, source.Items);
    }
}

// Usage: Type system enforces correct order
var draft = Order<IDraft>.CreateDraft(orderId, items);
var validated = draft.Validate();  // Returns Order<IValidated>
var processed = validated.Process();  // Returns Order<IProcessed>
var shipped = processed.Ship();  // Returns Order<IShipped>

// This won't compile:
// draft.Ship();  // Error: No method Ship on Order<IDraft>
// validated.Ship();  // Error: No method Ship on Order<IValidated>
```

## Builder Pattern with Phantom Types

```csharp
// Phantom types for required fields
public interface INameSet { }
public interface IEmailSet { }
public interface IPasswordSet { }

public class UserBuilder<TName, TEmail, TPassword>
{
    private string? _name;
    private string? _email;
    private string? _password;
    
    private UserBuilder() { }
    
    public static UserBuilder<object, object, object> Create() => new();
    
    public UserBuilder<INameSet, TEmail, TPassword> WithName(string name)
    {
        var builder = new UserBuilder<INameSet, TEmail, TPassword>
        {
            _name = name,
            _email = this._email,
            _password = this._password
        };
        return builder;
    }
    
    public UserBuilder<TName, IEmailSet, TPassword> WithEmail(string email)
    {
        var builder = new UserBuilder<TName, IEmailSet, TPassword>
        {
            _name = this._name,
            _email = email,
            _password = this._password
        };
        return builder;
    }
    
    public UserBuilder<TName, TEmail, IPasswordSet> WithPassword(string password)
    {
        var builder = new UserBuilder<TName, TEmail, IPasswordSet>
        {
            _name = this._name,
            _email = this._email,
            _password = password
        };
        return builder;
    }
}

// Build only available when all required fields are set
public static class UserBuilderExtensions
{
    public static User Build(
        this UserBuilder<INameSet, IEmailSet, IPasswordSet> builder)
    {
        return new User(builder._name!, builder._email!, builder._password!);
    }
}

// Usage
var user = UserBuilder
    .Create()
    .WithName("John")
    .WithEmail("john@example.com")
    .WithPassword("secret")
    .Build();  // Only compiles when all fields are set

// Won't compile:
// UserBuilder.Create().Build();  // Error: Build not available
// UserBuilder.Create().WithName("John").Build();  // Error: Email and password missing
```

## File Handles with Phantom Types

```csharp
public interface IFileState { }
public interface IClosed : IFileState { }
public interface IOpen : IFileState { }

public sealed class FileHandle<TState> : IDisposable where TState : IFileState
{
    private readonly string _path;
    private FileStream? _stream;
    
    private FileHandle(string path)
    {
        _path = path;
    }
    
    public static FileHandle<IClosed> Create(string path)
    {
        return new FileHandle<IClosed>(path);
    }
    
    public void Dispose()
    {
        _stream?.Dispose();
    }
}

public static class FileHandleExtensions
{
    public static FileHandle<IOpen> Open(this FileHandle<IClosed> handle)
    {
        var stream = File.OpenRead(handle._path);
        return new FileHandle<IOpen>(handle._path) { _stream = stream };
    }
    
    public static string Read(this FileHandle<IOpen> handle)
    {
        using var reader = new StreamReader(handle._stream!);
        return reader.ReadToEnd();
    }
    
    public static FileHandle<IClosed> Close(this FileHandle<IOpen> handle)
    {
        handle._stream?.Dispose();
        return new FileHandle<IClosed>(handle._path);
    }
}

// Usage
var file = FileHandle<IClosed>.Create("data.txt");
var openFile = file.Open();
var content = openFile.Read();  // Can only read when open
var closedFile = openFile.Close();

// Won't compile:
// file.Read();  // Error: No Read method on FileHandle<IClosed>
```

## Database Connection States

```csharp
public interface IConnectionState { }
public interface IDisconnected : IConnectionState { }
public interface IConnected : IConnectionState { }
public interface IInTransaction : IConnectionState { }

public class DatabaseConnection<TState> where TState : IConnectionState
{
    private readonly string _connectionString;
    
    internal DatabaseConnection(string connectionString)
    {
        _connectionString = connectionString;
    }
}

public static class DatabaseConnectionExtensions
{
    public static DatabaseConnection<IConnected> Connect(
        this DatabaseConnection<IDisconnected> conn)
    {
        // Open connection
        return new DatabaseConnection<IConnected>(conn._connectionString);
    }
    
    public static DatabaseConnection<IInTransaction> BeginTransaction(
        this DatabaseConnection<IConnected> conn)
    {
        // Start transaction
        return new DatabaseConnection<IInTransaction>(conn._connectionString);
    }
    
    public static void ExecuteQuery(
        this DatabaseConnection<IConnected> conn,
        string query)
    {
        // Can execute without transaction
    }
    
    public static void ExecuteQuery(
        this DatabaseConnection<IInTransaction> conn,
        string query)
    {
        // Can execute within transaction
    }
    
    public static DatabaseConnection<IConnected> Commit(
        this DatabaseConnection<IInTransaction> conn)
    {
        // Commit transaction
        return new DatabaseConnection<IConnected>(conn._connectionString);
    }
    
    public static DatabaseConnection<IConnected> Rollback(
        this DatabaseConnection<IInTransaction> conn)
    {
        // Rollback transaction
        return new DatabaseConnection<IConnected>(conn._connectionString);
    }
}

// Usage
var conn = new DatabaseConnection<IDisconnected>(connectionString);
var connected = conn.Connect();
var inTx = connected.BeginTransaction();
inTx.ExecuteQuery("INSERT INTO users...");
var committed = inTx.Commit();

// Won't compile:
// conn.ExecuteQuery("...");  // Error: Not connected
// conn.BeginTransaction();  // Error: Not connected
// connected.Commit();  // Error: No transaction to commit
```

## Why It's a Problem

1. **Runtime checks**: State validated at runtime, not compile time
2. **Invalid transitions**: Can call methods in wrong order
3. **Documentation burden**: Comments needed to explain valid state transitions
4. **Testing complexity**: Must test all invalid state transitions

## Symptoms

- State enums or boolean flags
- `if (state != ExpectedState) throw...` checks
- Comments like "Must be called after X"
- Methods that throw based on object state
- Tests for invalid state transitions

## Benefits

- **Compile-time safety**: Invalid state transitions don't compile
- **Self-documenting**: Type signature shows valid states
- **No runtime checks**: State validated by type system
- **Refactoring-safe**: Changing states breaks all invalid call sites

## Trade-offs

- **Type complexity**: More type parameters to track
- **API verbosity**: Method signatures longer
- **Learning curve**: Phantom types are advanced pattern
- **Generic constraints**: Can be complex to express

## See Also

- [Enforcing Call Order](./enforcing-call-order.md) — sequencing operations
- [Type-Safe Builder](./type-safe-builder.md) — required fields
- [Ghost States](./ghost-states.md) — contextual object states
- [Enum State Machine](./enum-state-machine.md) — state machines
