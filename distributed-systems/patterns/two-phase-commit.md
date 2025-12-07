# Two-Phase Commit (2PC)

> Distributed transaction protocol with a coordinator that ensures all participants commit or abort atomically—provides strong consistency at the cost of availability and performance.

## Problem

In a distributed system, a single business transaction may update data across multiple databases. If one update succeeds and another fails, data becomes inconsistent. Two-phase commit ensures all updates happen atomically—all succeed or all fail.

## The Protocol

```
Phase 1: Prepare               Phase 2: Commit/Abort
      
Coordinator                    Coordinator
     │                              │
     ├─► "Prepare" ──► Node A       ├─► "Commit" ──► Node A
     ├─► "Prepare" ──► Node B       ├─► "Commit" ──► Node B
     └─► "Prepare" ──► Node C       └─► "Commit" ──► Node C
            │                              │
            ▼                              ▼
    All vote "Yes"?               Nodes commit
            │                     transaction
            │
        Yes → Phase 2 (Commit)
        No  → Phase 2 (Abort)
```

## Example

### ❌ Without 2PC: Inconsistency Risk

```csharp
public class TransferService
{
    private readonly IAccountRepository _sourceRepo;
    private readonly IAccountRepository _targetRepo;
    
    public async Task TransferMoney(
        AccountId sourceId,
        AccountId targetId,
        Money amount)
    {
        // Debit source
        await _sourceRepo.DebitAsync(sourceId, amount);
        
        // What if crash happens here? Money disappears!
        
        // Credit target
        await _targetRepo.CreditAsync(targetId, amount);
        
        // What if this fails? Source debited but target not credited!
    }
}
```

### ✅ With Two-Phase Commit

```csharp
/// <summary>
/// Transaction participant interface.
/// </summary>
public interface ITransactionParticipant
{
    Task<bool> PrepareAsync(TransactionId transactionId);
    Task CommitAsync(TransactionId transactionId);
    Task AbortAsync(TransactionId transactionId);
}

/// <summary>
/// Transaction coordinator manages 2PC protocol.
/// </summary>
public class TransactionCoordinator
{
    private readonly ILogger<TransactionCoordinator> _logger;
    private readonly ITransactionLog _transactionLog;
    
    public async Task<Result<Unit, TransactionError>> ExecuteAsync(
        TransactionId transactionId,
        List<ITransactionParticipant> participants,
        Func<Task> operation)
    {
        // Log transaction start
        await _transactionLog.LogStartAsync(transactionId, participants);
        
        try
        {
            // Execute operation (creates local changes)
            await operation();
            
            // PHASE 1: PREPARE
            _logger.LogInformation(
                "Starting prepare phase for transaction {TransactionId}",
                transactionId);
            
            var prepareResults = await Task.WhenAll(
                participants.Select(p => p.PrepareAsync(transactionId)));
            
            // Check if all participants voted YES
            if (!prepareResults.All(result => result))
            {
                _logger.LogWarning(
                    "Transaction {TransactionId} aborted: Not all participants prepared",
                    transactionId);
                
                // PHASE 2: ABORT
                await AbortAllAsync(transactionId, participants);
                
                return Result<Unit, TransactionError>.Failure(
                    new TransactionError.PrepareFailed("One or more participants failed to prepare"));
            }
            
            // Log commit decision (point of no return!)
            await _transactionLog.LogCommitDecisionAsync(transactionId);
            
            // PHASE 2: COMMIT
            _logger.LogInformation(
                "Starting commit phase for transaction {TransactionId}",
                transactionId);
            
            await CommitAllAsync(transactionId, participants);
            
            // Log completion
            await _transactionLog.LogCompletedAsync(transactionId);
            
            return Result<Unit, TransactionError>.Success(Unit.Value);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, 
                "Transaction {TransactionId} failed",
                transactionId);
            
            // Abort all
            await AbortAllAsync(transactionId, participants);
            
            return Result<Unit, TransactionError>.Failure(
                new TransactionError.ExecutionFailed(ex.Message));
        }
    }
    
    private async Task CommitAllAsync(
        TransactionId transactionId,
        List<ITransactionParticipant> participants)
    {
        foreach (var participant in participants)
        {
            try
            {
                await participant.CommitAsync(transactionId);
            }
            catch (Exception ex)
            {
                // Commit must succeed—retry indefinitely
                _logger.LogError(ex, 
                    "Failed to commit participant in transaction {TransactionId}, will retry",
                    transactionId);
                
                // In production: queue for retry, alert operations
                throw;
            }
        }
    }
    
    private async Task AbortAllAsync(
        TransactionId transactionId,
        List<ITransactionParticipant> participants)
    {
        var abortTasks = participants.Select(async participant =>
        {
            try
            {
                await participant.AbortAsync(transactionId);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, 
                    "Failed to abort participant in transaction {TransactionId}",
                    transactionId);
                // Continue aborting others
            }
        });
        
        await Task.WhenAll(abortTasks);
    }
}

public abstract record TransactionError
{
    public sealed record PrepareFailed(string Message) : TransactionError;
    public sealed record ExecutionFailed(string Message) : TransactionError;
    public sealed record TimeoutError(string Message) : TransactionError;
}

/// <summary>
/// Account repository as transaction participant.
/// </summary>
public class AccountRepository : ITransactionParticipant
{
    private readonly IDatabase _database;
    private readonly ConcurrentDictionary<TransactionId, PendingTransaction> _pending = new();
    
    public async Task<bool> PrepareAsync(TransactionId transactionId)
    {
        // Get pending changes for this transaction
        if (!_pending.TryGetValue(transactionId, out var pending))
            return false;
        
        try
        {
            // Validate changes can be applied
            foreach (var change in pending.Changes)
            {
                var account = await _database.GetAccountAsync(change.AccountId);
                
                if (change.Type == ChangeType.Debit && account.Balance < change.Amount)
                {
                    // Insufficient funds—vote NO
                    return false;
                }
            }
            
            // Lock resources (pessimistic locking)
            foreach (var change in pending.Changes)
            {
                await _database.LockAccountAsync(change.AccountId);
            }
            
            // Vote YES
            pending.State = TransactionState.Prepared;
            return true;
        }
        catch
        {
            // Any error—vote NO
            return false;
        }
    }
    
    public async Task CommitAsync(TransactionId transactionId)
    {
        if (!_pending.TryGetValue(transactionId, out var pending))
            throw new InvalidOperationException("Transaction not found");
        
        if (pending.State != TransactionState.Prepared)
            throw new InvalidOperationException("Transaction not prepared");
        
        // Apply changes durably
        foreach (var change in pending.Changes)
        {
            await _database.ApplyChangeAsync(change);
        }
        
        // Release locks
        foreach (var change in pending.Changes)
        {
            await _database.UnlockAccountAsync(change.AccountId);
        }
        
        // Remove pending transaction
        _pending.TryRemove(transactionId, out _);
    }
    
    public async Task AbortAsync(TransactionId transactionId)
    {
        if (_pending.TryRemove(transactionId, out var pending))
        {
            // Release locks
            foreach (var change in pending.Changes)
            {
                await _database.UnlockAccountAsync(change.AccountId);
            }
        }
    }
    
    // Methods to stage changes before prepare
    public void StageDebit(TransactionId transactionId, AccountId accountId, Money amount)
    {
        var pending = _pending.GetOrAdd(transactionId, _ => new PendingTransaction
        {
            TransactionId = transactionId,
            State = TransactionState.Staged
        });
        
        pending.Changes.Add(new AccountChange
        {
            AccountId = accountId,
            Type = ChangeType.Debit,
            Amount = amount
        });
    }
    
    public void StageCredit(TransactionId transactionId, AccountId accountId, Money amount)
    {
        var pending = _pending.GetOrAdd(transactionId, _ => new PendingTransaction
        {
            TransactionId = transactionId,
            State = TransactionState.Staged
        });
        
        pending.Changes.Add(new AccountChange
        {
            AccountId = accountId,
            Type = ChangeType.Credit,
            Amount = amount
        });
    }
}

/// <summary>
/// Money transfer using 2PC.
/// </summary>
public class TransferService
{
    private readonly AccountRepository _accountRepo;
    private readonly TransactionCoordinator _coordinator;
    
    public async Task<Result<Unit, TransactionError>> TransferMoney(
        AccountId sourceId,
        AccountId targetId,
        Money amount)
    {
        var transactionId = TransactionId.New();
        
        // Stage changes
        _accountRepo.StageDebit(transactionId, sourceId, amount);
        _accountRepo.StageCredit(transactionId, targetId, amount);
        
        // Execute 2PC
        var participants = new List<ITransactionParticipant> { _accountRepo };
        
        return await _coordinator.ExecuteAsync(
            transactionId,
            participants,
            async () =>
            {
                // Additional validation or business logic
                await Task.CompletedTask;
            });
    }
}

public class PendingTransaction
{
    public required TransactionId TransactionId { get; init; }
    public TransactionState State { get; set; }
    public List<AccountChange> Changes { get; init; } = new();
}

public class AccountChange
{
    public required AccountId AccountId { get; init; }
    public required ChangeType Type { get; init; }
    public required Money Amount { get; init; }
}

public enum TransactionState
{
    Staged,
    Prepared,
    Committed,
    Aborted
}

public enum ChangeType
{
    Debit,
    Credit
}
```

## Transaction Log for Recovery

```csharp
/// <summary>
/// Persistent transaction log for crash recovery.
/// </summary>
public class TransactionLog : ITransactionLog
{
    private readonly IDatabase _database;
    
    public async Task LogStartAsync(
        TransactionId transactionId,
        List<ITransactionParticipant> participants)
    {
        await _database.ExecuteAsync(
            "INSERT INTO TransactionLog (TransactionId, State, Timestamp) " +
            "VALUES (@Id, @State, @Timestamp)",
            new
            {
                Id = transactionId.Value,
                State = "Started",
                Timestamp = DateTimeOffset.UtcNow
            });
    }
    
    public async Task LogCommitDecisionAsync(TransactionId transactionId)
    {
        // This is the critical log entry—once written, we must commit
        await _database.ExecuteAsync(
            "UPDATE TransactionLog SET State = @State, Timestamp = @Timestamp " +
            "WHERE TransactionId = @Id",
            new
            {
                Id = transactionId.Value,
                State = "CommitDecided",
                Timestamp = DateTimeOffset.UtcNow
            });
    }
    
    public async Task LogCompletedAsync(TransactionId transactionId)
    {
        await _database.ExecuteAsync(
            "UPDATE TransactionLog SET State = @State, Timestamp = @Timestamp " +
            "WHERE TransactionId = @Id",
            new
            {
                Id = transactionId.Value,
                State = "Completed",
                Timestamp = DateTimeOffset.UtcNow
            });
    }
    
    /// <summary>
    /// Recovery process on startup.
    /// </summary>
    public async Task RecoverAsync()
    {
        // Find transactions that decided to commit but didn't complete
        var incompleteTransactions = await _database.QueryAsync<TransactionId>(
            "SELECT TransactionId FROM TransactionLog " +
            "WHERE State = 'CommitDecided'");
        
        foreach (var transactionId in incompleteTransactions)
        {
            // Retry commit for each participant
            // (In production: load participant info from log)
            await RetryCommitAsync(transactionId);
        }
    }
    
    private async Task RetryCommitAsync(TransactionId transactionId)
    {
        // Implementation depends on stored participant info
        await Task.CompletedTask;
    }
}
```

## The Blocking Problem

2PC blocks if the coordinator crashes after prepare phase:

```csharp
/// <summary>
/// Timeout-based participant that unblocks if coordinator crashes.
/// </summary>
public class TimeoutParticipant : ITransactionParticipant
{
    private readonly TimeSpan _prepareTimeout = TimeSpan.FromSeconds(30);
    
    public async Task<bool> PrepareAsync(TransactionId transactionId)
    {
        // Prepare with timeout
        using var cts = new CancellationTokenSource(_prepareTimeout);
        
        try
        {
            // Do prepare work...
            await Task.Delay(100, cts.Token);
            return true;
        }
        catch (OperationCanceledException)
        {
            // Timeout—abort automatically
            await AbortAsync(transactionId);
            return false;
        }
    }
    
    public Task CommitAsync(TransactionId transactionId)
    {
        // Commit...
        return Task.CompletedTask;
    }
    
    public Task AbortAsync(TransactionId transactionId)
    {
        // Abort...
        return Task.CompletedTask;
    }
}
```

## When to Use 2PC

### ✅ Use When:

- **Strong consistency is critical**: Bank transfers, inventory updates
- **Few participants**: 2-3 databases at most
- **Low latency not required**: Can tolerate blocking
- **All participants support 2PC**: Have control over all systems

### ❌ Avoid When:

- **High availability needed**: 2PC reduces availability
- **Many participants**: Coordination cost grows with participants
- **Low latency required**: Prepare phase adds significant latency
- **External systems involved**: Can't coordinate with third parties

## Why It's a Problem (When Not Using 2PC for Strong Consistency)

1. **Inconsistency**: Partial updates leave system in invalid state
2. **Lost data**: Failures can lose committed changes
3. **Manual recovery**: Operators must fix inconsistencies
4. **Trust issues**: Users see inconsistent data

## Symptoms

- Data inconsistencies across databases
- Manual reconciliation processes
- Complex compensation logic
- Audit trail showing partial transactions

## Benefits

- **Strong consistency**: All-or-nothing guarantees
- **ACID properties**: Full transactional semantics
- **Simple reasoning**: Either complete success or complete failure

## Trade-offs

- **Reduced availability**: Blocking protocol
- **Poor performance**: Multiple round trips
- **Tight coupling**: All participants must be available
- **Scalability limits**: Coordination overhead increases with participants

## See Also

- [Saga Pattern](./saga-pattern.md) — alternative for long-running transactions
- [Eventual Consistency](./eventual-consistency.md) — accepting temporary inconsistency
- [CAP Theorem](./cap-theorem.md) — fundamental trade-offs
- [Event-Driven Architecture](./event-driven-architecture.md) — alternative coordination approach
