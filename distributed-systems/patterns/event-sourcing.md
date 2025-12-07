# Event Sourcing

> Store state as an append-only log of events rather than mutable records—enables time-travel debugging, audit trails, and event-driven architectures.

## Problem

Traditional databases store current state. Once you update a record, the previous state is lost. You can't answer questions like "What was the account balance on Tuesday?" or "Why did this order get cancelled?" Event sourcing solves this by storing every state change as an immutable event.

## Traditional State Storage vs. Event Sourcing

```
Traditional Database                Event Store
      
┌──────────────────┐              ┌──────────────────────┐
│ User Table       │              │ Event Log            │
├──────────────────┤              ├──────────────────────┤
│ id: 123          │              │ UserCreated          │
│ email: new@...   │              │   id: 123            │
│ status: active   │              │   email: old@...     │
└──────────────────┘              │   timestamp: T1      │
                                  │                      │
UPDATE loses                      │ EmailChanged         │
previous state!                   │   id: 123            │
                                  │   email: new@...     │
                                  │   timestamp: T2      │
                                  │                      │
                                  │ UserActivated        │
                                  │   id: 123            │
                                  │   timestamp: T3      │
                                  └──────────────────────┘
                                  
                                  Full history preserved!
                                  Can replay to any point!
```

## Example

### ❌ Traditional State Storage

```csharp
public class BankAccount
{
    public Guid Id { get; set; }
    public string AccountNumber { get; set; }
    public decimal Balance { get; set; }  // Only current balance
    public AccountStatus Status { get; set; }
}

public class BankAccountService
{
    private readonly IRepository<BankAccount> _repository;
    
    public async Task Deposit(Guid accountId, decimal amount)
    {
        var account = await _repository.GetByIdAsync(accountId);
        
        // Update balance—previous balance is lost
        account.Balance += amount;
        
        await _repository.UpdateAsync(account);
        
        // How did we get to this balance? Unknown!
    }
    
    public async Task Withdraw(Guid accountId, decimal amount)
    {
        var account = await _repository.GetByIdAsync(accountId);
        
        if (account.Balance < amount)
            throw new InsufficientFundsException();
        
        // Update balance—no audit trail
        account.Balance -= amount;
        
        await _repository.UpdateAsync(account);
    }
}
```

**Problems:**
- Lost history (can't answer "what happened?")
- No audit trail
- Can't replay events
- Can't derive different views from same data

### ✅ Event Sourcing

```csharp
/// <summary>
/// Events are immutable facts about what happened.
/// </summary>
public abstract record BankAccountEvent
{
    public Guid AccountId { get; init; }
    public DateTimeOffset Timestamp { get; init; }
    public long SequenceNumber { get; init; }
    
    public sealed record AccountOpened(
        Guid AccountId,
        string AccountNumber,
        DateTimeOffset Timestamp,
        long SequenceNumber) : BankAccountEvent;
    
    public sealed record MoneyDeposited(
        Guid AccountId,
        decimal Amount,
        string Description,
        DateTimeOffset Timestamp,
        long SequenceNumber) : BankAccountEvent;
    
    public sealed record MoneyWithdrawn(
        Guid AccountId,
        decimal Amount,
        string Description,
        DateTimeOffset Timestamp,
        long SequenceNumber) : BankAccountEvent;
    
    public sealed record AccountClosed(
        Guid AccountId,
        string Reason,
        DateTimeOffset Timestamp,
        long SequenceNumber) : BankAccountEvent;
}

/// <summary>
/// Event store interface.
/// </summary>
public interface IEventStore
{
    Task AppendAsync(Guid streamId, IEnumerable<BankAccountEvent> events);
    Task<IEnumerable<BankAccountEvent>> GetEventsAsync(Guid streamId);
    Task<IEnumerable<BankAccountEvent>> GetEventsAsync(Guid streamId, long fromSequence);
}

/// <summary>
/// Aggregate that can be reconstructed from events.
/// </summary>
public class BankAccount
{
    public Guid Id { get; private set; }
    public string AccountNumber { get; private set; } = "";
    public decimal Balance { get; private set; }
    public AccountStatus Status { get; private set; }
    public long Version { get; private set; }
    
    private readonly List<BankAccountEvent> _uncommittedEvents = new();
    
    // Reconstruct state from events
    public static BankAccount FromEvents(IEnumerable<BankAccountEvent> events)
    {
        var account = new BankAccount();
        
        foreach (var @event in events)
        {
            account.Apply(@event);
        }
        
        return account;
    }
    
    // Apply event to update state
    private void Apply(BankAccountEvent @event)
    {
        switch (@event)
        {
            case BankAccountEvent.AccountOpened opened:
                Id = opened.AccountId;
                AccountNumber = opened.AccountNumber;
                Status = AccountStatus.Open;
                Version = opened.SequenceNumber;
                break;
            
            case BankAccountEvent.MoneyDeposited deposited:
                Balance += deposited.Amount;
                Version = deposited.SequenceNumber;
                break;
            
            case BankAccountEvent.MoneyWithdrawn withdrawn:
                Balance -= withdrawn.Amount;
                Version = withdrawn.SequenceNumber;
                break;
            
            case BankAccountEvent.AccountClosed closed:
                Status = AccountStatus.Closed;
                Version = closed.SequenceNumber;
                break;
        }
    }
    
    // Commands produce events
    public void Deposit(decimal amount, string description)
    {
        if (Status != AccountStatus.Open)
            throw new InvalidOperationException("Account is not open");
        
        if (amount <= 0)
            throw new ArgumentException("Amount must be positive");
        
        var @event = new BankAccountEvent.MoneyDeposited(
            Id,
            amount,
            description,
            DateTimeOffset.UtcNow,
            Version + 1);
        
        Apply(@event);
        _uncommittedEvents.Add(@event);
    }
    
    public void Withdraw(decimal amount, string description)
    {
        if (Status != AccountStatus.Open)
            throw new InvalidOperationException("Account is not open");
        
        if (amount <= 0)
            throw new ArgumentException("Amount must be positive");
        
        if (Balance < amount)
            throw new InsufficientFundsException();
        
        var @event = new BankAccountEvent.MoneyWithdrawn(
            Id,
            amount,
            description,
            DateTimeOffset.UtcNow,
            Version + 1);
        
        Apply(@event);
        _uncommittedEvents.Add(@event);
    }
    
    public IEnumerable<BankAccountEvent> GetUncommittedEvents()
    {
        return _uncommittedEvents.AsReadOnly();
    }
    
    public void MarkEventsAsCommitted()
    {
        _uncommittedEvents.Clear();
    }
}

/// <summary>
/// Repository that works with event store.
/// </summary>
public class BankAccountRepository
{
    private readonly IEventStore _eventStore;
    
    public async Task<BankAccount> GetByIdAsync(Guid accountId)
    {
        // Load all events for this account
        var events = await _eventStore.GetEventsAsync(accountId);
        
        // Reconstruct state from events
        return BankAccount.FromEvents(events);
    }
    
    public async Task SaveAsync(BankAccount account)
    {
        // Get uncommitted events
        var events = account.GetUncommittedEvents();
        
        // Append to event store
        await _eventStore.AppendAsync(account.Id, events);
        
        // Mark as committed
        account.MarkEventsAsCommitted();
    }
}

/// <summary>
/// Service using event-sourced aggregate.
/// </summary>
public class BankAccountService
{
    private readonly BankAccountRepository _repository;
    
    public async Task DepositAsync(Guid accountId, decimal amount, string description)
    {
        // Load aggregate from events
        var account = await _repository.GetByIdAsync(accountId);
        
        // Execute command (produces events)
        account.Deposit(amount, description);
        
        // Save events
        await _repository.SaveAsync(account);
    }
    
    public async Task WithdrawAsync(Guid accountId, decimal amount, string description)
    {
        var account = await _repository.GetByIdAsync(accountId);
        account.Withdraw(amount, description);
        await _repository.SaveAsync(account);
    }
}
```

## Snapshots for Performance

Loading thousands of events is slow. Use snapshots:

```csharp
public class SnapshotStore
{
    public sealed record Snapshot(
        Guid AccountId,
        decimal Balance,
        AccountStatus Status,
        string AccountNumber,
        long Version,
        DateTimeOffset Timestamp);
    
    private readonly IEventStore _eventStore;
    private readonly ISnapshotRepository _snapshotRepository;
    
    public async Task<BankAccount> GetByIdAsync(Guid accountId)
    {
        // Try to get latest snapshot
        var snapshot = await _snapshotRepository.GetLatestAsync(accountId);
        
        if (snapshot != null)
        {
            // Load events since snapshot
            var events = await _eventStore.GetEventsAsync(
                accountId,
                fromSequence: snapshot.Version + 1);
            
            // Reconstruct from snapshot + recent events
            var account = BankAccount.FromSnapshot(snapshot);
            
            foreach (var @event in events)
            {
                account.Apply(@event);
            }
            
            return account;
        }
        
        // No snapshot, load all events
        var allEvents = await _eventStore.GetEventsAsync(accountId);
        return BankAccount.FromEvents(allEvents);
    }
    
    public async Task SaveAsync(BankAccount account)
    {
        var events = account.GetUncommittedEvents();
        await _eventStore.AppendAsync(account.Id, events);
        account.MarkEventsAsCommitted();
        
        // Create snapshot every 100 events
        if (account.Version % 100 == 0)
        {
            var snapshot = new Snapshot(
                account.Id,
                account.Balance,
                account.Status,
                account.AccountNumber,
                account.Version,
                DateTimeOffset.UtcNow);
            
            await _snapshotRepository.SaveAsync(snapshot);
        }
    }
}
```

## Projections: Building Read Models

Events are the source of truth. Build specialized read models:

```csharp
/// <summary>
/// Read model for account balance queries.
/// </summary>
public class AccountBalanceProjection : IEventHandler<BankAccountEvent>
{
    private readonly IBalanceRepository _balanceRepository;
    
    public async Task HandleAsync(BankAccountEvent @event)
    {
        switch (@event)
        {
            case BankAccountEvent.AccountOpened opened:
                await _balanceRepository.CreateAsync(new AccountBalance
                {
                    AccountId = opened.AccountId,
                    Balance = 0,
                    LastUpdated = opened.Timestamp
                });
                break;
            
            case BankAccountEvent.MoneyDeposited deposited:
                await _balanceRepository.IncrementAsync(
                    deposited.AccountId,
                    deposited.Amount,
                    deposited.Timestamp);
                break;
            
            case BankAccountEvent.MoneyWithdrawn withdrawn:
                await _balanceRepository.DecrementAsync(
                    withdrawn.AccountId,
                    withdrawn.Amount,
                    withdrawn.Timestamp);
                break;
        }
    }
}

/// <summary>
/// Read model for transaction history queries.
/// </summary>
public class TransactionHistoryProjection : IEventHandler<BankAccountEvent>
{
    private readonly ITransactionRepository _transactionRepository;
    
    public async Task HandleAsync(BankAccountEvent @event)
    {
        switch (@event)
        {
            case BankAccountEvent.MoneyDeposited deposited:
                await _transactionRepository.AddAsync(new Transaction
                {
                    AccountId = deposited.AccountId,
                    Type = TransactionType.Deposit,
                    Amount = deposited.Amount,
                    Description = deposited.Description,
                    Timestamp = deposited.Timestamp
                });
                break;
            
            case BankAccountEvent.MoneyWithdrawn withdrawn:
                await _transactionRepository.AddAsync(new Transaction
                {
                    AccountId = withdrawn.AccountId,
                    Type = TransactionType.Withdrawal,
                    Amount = withdrawn.Amount,
                    Description = withdrawn.Description,
                    Timestamp = withdrawn.Timestamp
                });
                break;
        }
    }
}
```

## Time Travel: Replay to Any Point

```csharp
public class TimeTravelQuery
{
    private readonly IEventStore _eventStore;
    
    /// <summary>
    /// Get account balance at specific point in time.
    /// </summary>
    public async Task<decimal> GetBalanceAtAsync(Guid accountId, DateTimeOffset timestamp)
    {
        var events = await _eventStore.GetEventsAsync(accountId);
        
        // Filter events up to timestamp
        var eventsUpToTimestamp = events
            .Where(e => e.Timestamp <= timestamp)
            .OrderBy(e => e.SequenceNumber);
        
        // Reconstruct state
        var account = BankAccount.FromEvents(eventsUpToTimestamp);
        
        return account.Balance;
    }
    
    /// <summary>
    /// Get all transactions in date range.
    /// </summary>
    public async Task<List<Transaction>> GetTransactionsAsync(
        Guid accountId,
        DateTimeOffset from,
        DateTimeOffset to)
    {
        var events = await _eventStore.GetEventsAsync(accountId);
        
        return events
            .Where(e => e.Timestamp >= from && e.Timestamp <= to)
            .Select(e => e switch
            {
                BankAccountEvent.MoneyDeposited deposited => new Transaction
                {
                    Type = TransactionType.Deposit,
                    Amount = deposited.Amount,
                    Description = deposited.Description,
                    Timestamp = deposited.Timestamp
                },
                BankAccountEvent.MoneyWithdrawn withdrawn => new Transaction
                {
                    Type = TransactionType.Withdrawal,
                    Amount = withdrawn.Amount,
                    Description = withdrawn.Description,
                    Timestamp = withdrawn.Timestamp
                },
                _ => null
            })
            .Where(t => t != null)
            .ToList()!;
    }
}
```

## Event Store Implementation

```csharp
public class EventStore : IEventStore
{
    private readonly IDatabase _database;
    
    public async Task AppendAsync(Guid streamId, IEnumerable<BankAccountEvent> events)
    {
        foreach (var @event in events)
        {
            await _database.ExecuteAsync(
                "INSERT INTO Events (StreamId, SequenceNumber, EventType, Data, Timestamp) " +
                "VALUES (@StreamId, @SequenceNumber, @EventType, @Data, @Timestamp)",
                new
                {
                    StreamId = streamId,
                    @event.SequenceNumber,
                    EventType = @event.GetType().Name,
                    Data = JsonSerializer.Serialize(@event),
                    @event.Timestamp
                });
        }
    }
    
    public async Task<IEnumerable<BankAccountEvent>> GetEventsAsync(Guid streamId)
    {
        var rows = await _database.QueryAsync(
            "SELECT EventType, Data FROM Events " +
            "WHERE StreamId = @StreamId " +
            "ORDER BY SequenceNumber",
            new { StreamId = streamId });
        
        return rows.Select(row =>
            JsonSerializer.Deserialize<BankAccountEvent>(row.Data)!);
    }
    
    public async Task<IEnumerable<BankAccountEvent>> GetEventsAsync(
        Guid streamId,
        long fromSequence)
    {
        var rows = await _database.QueryAsync(
            "SELECT EventType, Data FROM Events " +
            "WHERE StreamId = @StreamId AND SequenceNumber >= @FromSequence " +
            "ORDER BY SequenceNumber",
            new { StreamId = streamId, FromSequence = fromSequence });
        
        return rows.Select(row =>
            JsonSerializer.Deserialize<BankAccountEvent>(row.Data)!);
    }
}
```

## Why It's a Problem (Not Using Event Sourcing)

1. **Lost history**: Can't answer "what happened?" or "why?"
2. **No audit trail**: Compliance and debugging challenges
3. **Can't replay**: No way to reconstruct past states
4. **Single view**: Database schema forces one representation

## Symptoms

- Audit requirements need complex triggers and history tables
- Can't answer questions about past state
- Debugging requires logs (which may be incomplete)
- Schema changes require migrations

## Benefits

- **Complete audit trail**: Every state change is recorded
- **Time travel**: Replay events to any point in time
- **Multiple views**: Build different read models from same events
- **Event-driven**: Natural fit for event-driven architectures
- **Debugging**: Full history makes debugging easier

## See Also

- [CQRS](./cqrs.md) — separate read and write models
- [Event-Driven Architecture](./event-driven-architecture.md) — events for communication
- [Saga Pattern](./saga-pattern.md) — events for coordination
- [Eventual Consistency](./eventual-consistency.md) — projections update eventually
