# Type-Level State Machines

> Using enums or booleans to track state allows invalid transitions—use sealed class hierarchies to make illegal state transitions uncompilable.

## Problem

Traditional state machines use enum fields and switch statements to track state. This allows invalid state transitions, requires runtime validation, and provides no compile-time guarantees about state validity. Type-level state machines encode states as distinct types, making invalid transitions impossible to express.

## Example

### ❌ Before (Enum-Based State Machine)

```csharp
public enum ConnectionState
{
    Disconnected,
    Connecting,
    Connected,
    Disconnecting,
    Failed
}

public class DatabaseConnection
{
    public ConnectionState State { get; private set; } = ConnectionState.Disconnected;
    public string? ConnectionString { get; private set; }
    public SqlConnection? Connection { get; private set; }
    public Exception? LastError { get; private set; }
    
    public void Connect(string connectionString)
    {
        // Runtime check: can only connect when disconnected
        if (State != ConnectionState.Disconnected)
            throw new InvalidOperationException($"Cannot connect from state {State}");
        
        State = ConnectionState.Connecting;
        ConnectionString = connectionString;
        
        try
        {
            Connection = new SqlConnection(connectionString);
            Connection.Open();
            State = ConnectionState.Connected;
        }
        catch (Exception ex)
        {
            LastError = ex;
            State = ConnectionState.Failed;
            throw;
        }
    }
    
    public void Disconnect()
    {
        // Runtime check: can only disconnect when connected
        if (State != ConnectionState.Connected)
            throw new InvalidOperationException($"Cannot disconnect from state {State}");
        
        State = ConnectionState.Disconnecting;
        Connection?.Close();
        Connection = null;
        State = ConnectionState.Disconnected;
    }
    
    public void ExecuteQuery(string sql)
    {
        // Must check state before executing
        if (State != ConnectionState.Connected)
            throw new InvalidOperationException("Not connected");
        
        if (Connection is null)
            throw new InvalidOperationException("Connection is null");
        
        // Execute query
        using var cmd = Connection.CreateCommand();
        cmd.CommandText = sql;
        cmd.ExecuteNonQuery();
    }
}

// Problems:
// 1. Can set any state at any time (compiles but fails at runtime)
// 2. Fields like Connection might be null or non-null depending on state
// 3. Must remember to check state before every operation
// 4. State transitions validated at runtime only

var conn = new DatabaseConnection();
conn.Disconnect();  // Compiles but throws: can't disconnect when disconnected
conn.ExecuteQuery("SELECT 1");  // Compiles but throws: not connected
```

### ✅ After (Type-Level State Machine)

```csharp
/// <summary>
/// Base class for all connection states.
/// Sealed to prevent external extension.
/// </summary>
public abstract record DatabaseConnection
{
    // Factory method creates initial state
    public static DisconnectedConnection Disconnected() => new();
}

/// <summary>
/// Disconnected state: can only transition to Connecting.
/// No connection data available in this state.
/// </summary>
public sealed record DisconnectedConnection : DatabaseConnection
{
    // Transition: Disconnected → Connecting
    public ConnectingConnection Connect(string connectionString)
    {
        return new ConnectingConnection(connectionString);
    }
}

/// <summary>
/// Connecting state: async transition to Connected or Failed.
/// Has connection string but no active connection.
/// </summary>
public sealed record ConnectingConnection : DatabaseConnection
{
    public string ConnectionString { get; }
    
    internal ConnectingConnection(string connectionString)
    {
        ConnectionString = connectionString;
    }
    
    // Transition: Connecting → Connected (success) or Failed (error)
    public async Task<DatabaseConnection> AwaitConnection()
    {
        var connection = new SqlConnection(this.ConnectionString);
        
        try
        {
            await connection.OpenAsync();
            return new ConnectedConnection(this.ConnectionString, connection);
        }
        catch (Exception ex)
        {
            return new FailedConnection(this.ConnectionString, ex);
        }
    }
}

/// <summary>
/// Connected state: has active connection, can execute queries.
/// Can only transition to Disconnecting.
/// </summary>
public sealed record ConnectedConnection : DatabaseConnection
{
    public string ConnectionString { get; }
    public SqlConnection Connection { get; }
    
    internal ConnectedConnection(string connectionString, SqlConnection connection)
    {
        ConnectionString = connectionString;
        Connection = connection;
    }
    
    // Operations only available in Connected state
    public async Task<int> ExecuteQuery(string sql)
    {
        // No state check needed—type guarantees connection is ready
        using var cmd = this.Connection.CreateCommand();
        cmd.CommandText = sql;
        return await cmd.ExecuteNonQueryAsync();
    }
    
    public async Task<T> ExecuteScalar<T>(string sql)
    {
        using var cmd = this.Connection.CreateCommand();
        cmd.CommandText = sql;
        var result = await cmd.ExecuteScalarAsync();
        return (T)result;
    }
    
    // Transition: Connected → Disconnecting
    public DisconnectingConnection Disconnect()
    {
        return new DisconnectingConnection(this.Connection);
    }
}

/// <summary>
/// Disconnecting state: cleaning up resources.
/// Transitions to Disconnected when complete.
/// </summary>
public sealed record DisconnectingConnection : DatabaseConnection
{
    SqlConnection Connection { get; }
    
    internal DisconnectingConnection(SqlConnection connection)
    {
        Connection = connection;
    }
    
    // Transition: Disconnecting → Disconnected
    public async Task<DisconnectedConnection> AwaitDisconnection()
    {
        await this.Connection.CloseAsync();
        this.Connection.Dispose();
        return new DisconnectedConnection();
    }
}

/// <summary>
/// Failed state: connection attempt failed.
/// Has error information, can retry or give up.
/// </summary>
public sealed record FailedConnection : DatabaseConnection
{
    public string ConnectionString { get; }
    public Exception Error { get; }
    
    internal FailedConnection(string connectionString, Exception error)
    {
        ConnectionString = connectionString;
        Error = error;
    }
    
    // Transition: Failed → Connecting (retry)
    public ConnectingConnection Retry()
    {
        return new ConnectingConnection(this.ConnectionString);
    }
    
    // Transition: Failed → Disconnected (give up)
    public DisconnectedConnection GiveUp()
    {
        return new DisconnectedConnection();
    }
}

// Usage: Type system enforces valid transitions
public class DatabaseService
{
    async Task UseDatabase()
    {
        // Start disconnected
        DisconnectedConnection disconnected = DatabaseConnection.Disconnected();
        
        // Connect (type changes)
        ConnectingConnection connecting = disconnected.Connect("Server=localhost;...");
        
        // Await connection (might succeed or fail)
        DatabaseConnection result = await connecting.AwaitConnection();
        
        // Pattern match on result type
        switch (result)
        {
            case ConnectedConnection connected:
                // Can only execute queries when Connected
                await connected.ExecuteQuery("INSERT INTO Logs VALUES ('Started')");
                
                // Disconnect when done
                DisconnectingConnection disconnecting = connected.Disconnect();
                DisconnectedConnection done = await disconnecting.AwaitDisconnection();
                break;
            
            case FailedConnection failed:
                Console.WriteLine($"Connection failed: {failed.Error.Message}");
                
                // Can retry or give up
                ConnectingConnection retry = failed.Retry();
                // ... or ...
                DisconnectedConnection gaveUp = failed.GiveUp();
                break;
        }
    }
}

// Impossible operations don't compile:
// disconnected.ExecuteQuery("SELECT 1");  // ❌ Error: ExecuteQuery not defined on Disconnected
// disconnected.Disconnect();  // ❌ Error: Disconnect not defined on Disconnected
// connecting.ExecuteQuery("SELECT 1");  // ❌ Error: ExecuteQuery not defined on Connecting
```

## HTTP Request/Response State Machine

```csharp
public abstract record HttpRequest;

/// <summary>
/// Request hasn't been sent yet—can modify headers and body.
/// </summary>
public sealed record UnsentRequest : HttpRequest
{
    public string Url { get; }
    public Dictionary<string, string> Headers { get; } = new();
    public string? Body { get; init; }
    
    public UnsentRequest(string url)
    {
        Url = url;
    }
    
    public UnsentRequest WithHeader(string name, string value)
    {
        var newRequest = this with { };
        newRequest.Headers[name] = value;
        return newRequest;
    }
    
    public SentRequest Send()
    {
        // Transition: Unsent → Sent
        return new SentRequest(this.Url, this.Headers, this.Body);
    }
}

/// <summary>
/// Request has been sent—waiting for response.
/// Cannot modify headers anymore.
/// </summary>
public sealed record SentRequest : HttpRequest
{
    public string Url { get; }
    public IReadOnlyDictionary<string, string> Headers { get; }
    public string? Body { get; }
    
    internal SentRequest(
        string url,
        Dictionary<string, string> headers,
        string? body)
    {
        Url = url;
        Headers = headers;
        Body = body;
    }
    
    public async Task<HttpRequest> AwaitResponse()
    {
        try
        {
            using var client = new HttpClient();
            var response = await client.GetAsync(this.Url);
            
            if (response.IsSuccessStatusCode)
            {
                var content = await response.Content.ReadAsStringAsync();
                return new SuccessResponse(
                    this.Url,
                    (int)response.StatusCode,
                    content);
            }
            else
            {
                return new ErrorResponse(
                    this.Url,
                    (int)response.StatusCode,
                    await response.Content.ReadAsStringAsync());
            }
        }
        catch (Exception ex)
        {
            return new FailedRequest(this.Url, ex);
        }
    }
}

/// <summary>
/// Request succeeded—has response data.
/// </summary>
public sealed record SuccessResponse : HttpRequest
{
    public string Url { get; }
    public int StatusCode { get; }
    public string Content { get; }
    
    internal SuccessResponse(string url, int statusCode, string content)
    {
        Url = url;
        StatusCode = statusCode;
        Content = content;
    }
}

/// <summary>
/// Request returned error status code.
/// </summary>
public sealed record ErrorResponse : HttpRequest
{
    public string Url { get; }
    public int StatusCode { get; }
    public string ErrorContent { get; }
    
    internal ErrorResponse(string url, int statusCode, string errorContent)
    {
        Url = url;
        StatusCode = statusCode;
        ErrorContent = errorContent;
    }
    
    public UnsentRequest Retry() => new(this.Url);
}

/// <summary>
/// Request failed before getting response.
/// </summary>
public sealed record FailedRequest : HttpRequest
{
    public string Url { get; }
    public Exception Exception { get; }
    
    internal FailedRequest(string url, Exception exception)
    {
        Url = url;
        Exception = exception;
    }
    
    public UnsentRequest Retry() => new(this.Url);
}

// Usage
async Task MakeRequest()
{
    var request = new UnsentRequest("https://api.example.com/data")
        .WithHeader("Authorization", "Bearer token")
        .WithHeader("Accept", "application/json");
    
    var sent = request.Send();
    var result = await sent.AwaitResponse();
    
    switch (result)
    {
        case SuccessResponse success:
            Console.WriteLine($"Success: {success.Content}");
            break;
        
        case ErrorResponse error:
            Console.WriteLine($"Error {error.StatusCode}: {error.ErrorContent}");
            var retry = error.Retry();
            break;
        
        case FailedRequest failed:
            Console.WriteLine($"Failed: {failed.Exception.Message}");
            break;
    }
}
```

## File Processing State Machine

```csharp
public abstract record FileProcessor;

public sealed record FileNotLoaded : FileProcessor
{
    public FileInfo FileInfo { get; }
    
    public FileNotLoaded(string path)
    {
        FileInfo = new FileInfo(path);
    }
    
    public async Task<FileProcessor> Load()
    {
        try
        {
            var content = await File.ReadAllTextAsync(FileInfo.FullName);
            return new FileLoaded(FileInfo, content);
        }
        catch (Exception ex)
        {
            return new FileLoadFailed(FileInfo, ex);
        }
    }
}

public sealed record FileLoaded : FileProcessor
{
    public FileInfo FileInfo { get; }
    public string Content { get; }
    
    internal FileLoaded(FileInfo fileInfo, string content)
    {
        FileInfo = fileInfo;
        Content = content;
    }
    
    public FileProcessor Validate()
    {
        if (string.IsNullOrWhiteSpace(Content))
            return new FileInvalid(FileInfo, "File is empty");
        
        if (Content.Length > 1_000_000)
            return new FileInvalid(FileInfo, "File too large");
        
        return new FileValidated(FileInfo, Content);
    }
}

public sealed record FileValidated : FileProcessor
{
    public FileInfo FileInfo { get; }
    public string Content { get; }
    
    internal FileValidated(FileInfo fileInfo, string content)
    {
        FileInfo = fileInfo;
        Content = content;
    }
    
    // Can only process validated files
    public FileProcessed Process()
    {
        var processed = Content.ToUpperInvariant();
        return new FileProcessed(FileInfo, processed);
    }
}

public sealed record FileProcessed : FileProcessor
{
    public FileInfo FileInfo { get; }
    public string ProcessedContent { get; }
    
    internal FileProcessed(FileInfo fileInfo, string processedContent)
    {
        FileInfo = fileInfo;
        ProcessedContent = processedContent;
    }
    
    public async Task<FileSaved> Save()
    {
        await File.WriteAllTextAsync(FileInfo.FullName, ProcessedContent);
        return new FileSaved(FileInfo);
    }
}

public sealed record FileSaved : FileProcessor
{
    public FileInfo FileInfo { get; }
    
    internal FileSaved(FileInfo fileInfo)
    {
        FileInfo = fileInfo;
    }
}

public sealed record FileInvalid : FileProcessor
{
    public FileInfo FileInfo { get; }
    public string ValidationError { get; }
    
    internal FileInvalid(FileInfo fileInfo, string error)
    {
        FileInfo = fileInfo;
        ValidationError = error;
    }
}

public sealed record FileLoadFailed : FileProcessor
{
    public FileInfo FileInfo { get; }
    public Exception Exception { get; }
    
    internal FileLoadFailed(FileInfo fileInfo, Exception exception)
    {
        FileInfo = fileInfo;
        Exception = exception;
    }
}
```

## Why It's a Problem

1. **Runtime state validation**: State checked at runtime, not compile time
2. **Invalid transitions possible**: Any state change can be expressed
3. **Scattered validation**: State checks in every method
4. **Incomplete state data**: Fields may be null/invalid depending on state

## Symptoms

- Enum fields tracking state
- `if (state != ExpectedState) throw` checks
- Nullable fields that are sometimes required
- Comments like "Only valid in state X"
- Runtime state transition errors

## Benefits

- **Compile-time state validation**: Invalid transitions don't compile
- **Type-safe state data**: Each state has appropriate fields
- **Exhaustive pattern matching**: Compiler ensures all states handled
- **Self-documenting**: Types show valid transitions
- **No runtime checks**: State validity guaranteed by type system

## Trade-offs

- **More types**: One type per state
- **Verbose transitions**: Must explicitly handle each state
- **Learning curve**: Team must understand state machines as types
- **Refactoring effort**: Converting enum-based state machines requires significant work

## See Also

- [Phantom Types](./phantom-types.md) — encoding state in type parameters
- [Enum State Machine](./enum-state-machine.md) — refactoring enum state machines
- [Type Witnesses](./type-witnesses.md) — proofs of conditions
- [Enforcing Call Order](./enforcing-call-order.md) — sequencing operations
