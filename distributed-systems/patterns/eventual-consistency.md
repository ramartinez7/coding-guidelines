# Eventual Consistency

> Accept temporary inconsistency across nodes for better availability and performance—eventually all nodes converge to the same state.

## Problem

Strong consistency requires coordination across nodes, which is slow and reduces availability. Most applications can tolerate brief periods where different nodes have different data, as long as they eventually converge.

## The Trade-off

```
Strong Consistency              Eventual Consistency
      
┌────────────────┐              ┌────────────────┐
│  Write to A    │              │  Write to A    │
└───────┬────────┘              └───────┬────────┘
        │                               │
        │ Wait for sync                 │ Return immediately
        │ (slow)                        │ (fast)
        ▼                               ▼
┌────────────────┐              ┌────────────────┐
│  Sync to B     │              │ Async to B     │
└───────┬────────┘              │ (background)   │
        │                       └────────────────┘
        │ Wait for sync                 │
        │ (slow)                        │
        ▼                               ▼
┌────────────────┐              
│  Sync to C     │              Nodes may disagree
└───────┬────────┘              temporarily, but
        │                       eventually converge
        ▼
  All synced!
  (but slow)
```

## Example

### ❌ Strong Consistency: Slow and Fragile

```csharp
public class UserProfileService
{
    private readonly IDatabase _primaryDb;
    private readonly IDatabase _replicaDb;
    private readonly ICache _cache;
    private readonly ISearchIndex _searchIndex;
    
    public async Task UpdateProfile(UserId userId, UserProfile profile)
    {
        // Wait for everything to complete before returning
        await _primaryDb.UpdateAsync(userId, profile);
        await _replicaDb.UpdateAsync(userId, profile);  // Blocks on replication
        await _cache.InvalidateAsync(userId);            // Blocks on cache
        await _searchIndex.UpdateAsync(userId, profile);  // Blocks on index
        
        // If any step fails, the whole operation fails
        // If any system is slow, the whole operation is slow
    }
}
```

### ✅ Eventual Consistency: Fast and Resilient

```csharp
/// <summary>
/// Event representing a profile update.
/// </summary>
public sealed record UserProfileUpdated(
    UserId UserId,
    UserProfile Profile,
    DateTimeOffset Timestamp,
    CorrelationId CorrelationId);

/// <summary>
/// Service that updates the primary database and publishes an event.
/// Other systems react asynchronously.
/// </summary>
public class UserProfileService
{
    private readonly IDatabase _primaryDb;
    private readonly IEventBus _eventBus;
    
    public async Task<Result<Unit, UpdateError>> UpdateProfile(
        UserId userId,
        UserProfile profile)
    {
        // Update primary database
        await _primaryDb.UpdateAsync(userId, profile);
        
        // Publish event and return immediately
        var @event = new UserProfileUpdated(
            userId,
            profile,
            DateTimeOffset.UtcNow,
            CorrelationId.New());
        
        await _eventBus.PublishAsync(@event);
        
        // Done! Other systems update asynchronously
        return Result<Unit, UpdateError>.Success(Unit.Value);
    }
}

/// <summary>
/// Cache invalidation happens asynchronously.
/// </summary>
public class CacheInvalidationHandler : IEventHandler<UserProfileUpdated>
{
    private readonly ICache _cache;
    private readonly ILogger<CacheInvalidationHandler> _logger;
    
    public async Task HandleAsync(UserProfileUpdated @event)
    {
        try
        {
            await _cache.InvalidateAsync(@event.UserId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, 
                "Failed to invalidate cache for user {UserId}", 
                @event.UserId);
            // Don't fail the original operation—retry later
        }
    }
}

/// <summary>
/// Search index updates asynchronously.
/// </summary>
public class SearchIndexHandler : IEventHandler<UserProfileUpdated>
{
    private readonly ISearchIndex _searchIndex;
    private readonly ILogger<SearchIndexHandler> _logger;
    
    public async Task HandleAsync(UserProfileUpdated @event)
    {
        try
        {
            await _searchIndex.UpdateAsync(@event.UserId, @event.Profile);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, 
                "Failed to update search index for user {UserId}", 
                @event.UserId);
            // Retry with exponential backoff
            throw;  // Message bus will retry
        }
    }
}

/// <summary>
/// Read replica synchronizes asynchronously.
/// </summary>
public class ReplicaSyncHandler : IEventHandler<UserProfileUpdated>
{
    private readonly IDatabase _replicaDb;
    
    public async Task HandleAsync(UserProfileUpdated @event)
    {
        await _replicaDb.UpdateAsync(@event.UserId, @event.Profile);
    }
}
```

## Benefits

### 1. Performance

Operations complete quickly without waiting for synchronization.

```csharp
// Strong: 500ms (wait for all systems)
// Eventual: 50ms (update primary + publish event)
```

### 2. Availability

If downstream systems fail, the operation still succeeds.

```csharp
// Strong: Search index down? Operation fails!
// Eventual: Search index down? Operation succeeds, index updates when it recovers
```

### 3. Independent Scaling

Each component scales independently.

```csharp
// Strong: Slowest component limits throughput
// Eventual: Each component processes at its own pace
```

## Handling Temporary Inconsistency

### Show Optimistic UI

```csharp
public class UserProfileController : ControllerBase
{
    [HttpPut("users/{userId}/profile")]
    public async Task<IActionResult> UpdateProfile(
        UserId userId,
        UserProfileDto profileDto)
    {
        var result = await _profileService.UpdateProfile(userId, profileDto.ToModel());
        
        if (result.IsFailure)
        {
            return BadRequest(result.Error);
        }
        
        // Return success immediately
        // Client can update UI optimistically
        return Ok(new
        {
            Message = "Profile updated successfully",
            Note = "Changes may take a moment to appear in search results"
        });
    }
}
```

### Version Vectors for Conflict Resolution

```csharp
/// <summary>
/// Profile with version vector for conflict resolution.
/// </summary>
public sealed record VersionedProfile
{
    public required UserId UserId { get; init; }
    public required UserProfile Profile { get; init; }
    public required Dictionary<NodeId, long> VersionVector { get; init; }
    
    /// <summary>
    /// Determines if this version happened before another.
    /// </summary>
    public bool HappenedBefore(VersionedProfile other)
    {
        return this.VersionVector.All(kvp =>
            other.VersionVector.TryGetValue(kvp.Key, out var otherVersion) &&
            kvp.Value <= otherVersion);
    }
    
    /// <summary>
    /// Merges two concurrent updates using last-write-wins.
    /// </summary>
    public static VersionedProfile Merge(
        VersionedProfile a,
        VersionedProfile b)
    {
        // If one happened before the other, use the later one
        if (a.HappenedBefore(b))
            return b;
        if (b.HappenedBefore(a))
            return a;
        
        // Concurrent updates—use conflict resolution strategy
        // (last-write-wins, merge fields, or application-specific logic)
        var mergedVector = a.VersionVector.Keys
            .Union(b.VersionVector.Keys)
            .ToDictionary(
                key => key,
                key => Math.Max(
                    a.VersionVector.GetValueOrDefault(key),
                    b.VersionVector.GetValueOrDefault(key)));
        
        return new VersionedProfile
        {
            UserId = a.UserId,
            Profile = MergeProfiles(a.Profile, b.Profile),
            VersionVector = mergedVector
        };
    }
    
    private static UserProfile MergeProfiles(UserProfile a, UserProfile b)
    {
        // Application-specific merge logic
        // Example: use field-level last-write-wins
        return new UserProfile
        {
            Name = a.Name.Timestamp > b.Name.Timestamp ? a.Name : b.Name,
            Email = a.Email.Timestamp > b.Email.Timestamp ? a.Email : b.Email,
            Bio = a.Bio.Timestamp > b.Bio.Timestamp ? a.Bio : b.Bio
        };
    }
}
```

### Idempotent Updates

Ensure updates can be applied multiple times safely.

```csharp
public class IdempotentSearchIndexHandler : IEventHandler<UserProfileUpdated>
{
    private readonly ISearchIndex _searchIndex;
    private readonly IIdempotencyStore _idempotency;
    
    public async Task HandleAsync(UserProfileUpdated @event)
    {
        // Check if we've already processed this event
        var key = new IdempotencyKey($"search-index-{@event.UserId}-{@event.Timestamp}");
        
        if (await _idempotency.HasProcessedAsync(key))
        {
            // Already processed—skip
            return;
        }
        
        // Update search index
        await _searchIndex.UpdateAsync(@event.UserId, @event.Profile);
        
        // Mark as processed
        await _idempotency.MarkProcessedAsync(key);
    }
}
```

## When NOT to Use Eventual Consistency

### Critical Invariants

```csharp
// ❌ Don't use eventual consistency for:

// Bank transfers (balance must be consistent)
public void TransferMoney(AccountId from, AccountId to, Money amount)
{
    // Must be atomic—cannot have eventual consistency here
    _db.Debit(from, amount);
    _db.Credit(to, amount);
}

// Seat reservations (cannot double-book)
public void ReserveSeat(FlightId flightId, SeatNumber seat, UserId userId)
{
    // Must check availability atomically
    if (_seats.IsAvailable(flightId, seat))
        _seats.Reserve(flightId, seat, userId);
}

// Authentication (must validate immediately)
public bool Login(Username username, Password password)
{
    // Cannot accept "eventually correct" authentication
    return _auth.ValidateCredentials(username, password);
}
```

### Use Strong Consistency When:

1. **Money is involved**: Financial transactions, billing, payments
2. **Inventory is limited**: Seat reservations, product stock, ticket sales
3. **Access control**: Authentication, authorization, permissions
4. **Business invariants are critical**: Cannot be violated even temporarily

## Why It's a Problem (Not Using Eventual Consistency)

1. **Poor performance**: Waiting for synchronization slows every operation
2. **Reduced availability**: One slow/down system blocks everything
3. **Tight coupling**: Changes to one system affect all systems
4. **Cascading failures**: Failures propagate across system boundaries

## Symptoms

- Slow write operations that block on multiple systems
- Timeouts when downstream systems are slow
- Operations fail when non-critical systems are down
- High latency spikes during traffic bursts

## Benefits

- **Fast operations**: Return immediately without waiting for propagation
- **High availability**: Downstream failures don't block operations
- **Independent scaling**: Each component scales based on its needs
- **Resilience**: System continues working even when parts fail

## See Also

- [CAP Theorem](./cap-theorem.md) — fundamental trade-offs
- [Event Sourcing](./event-sourcing.md) — event log as source of truth
- [CQRS](./cqrs.md) — separate read and write models
- [Saga Pattern](./saga-pattern.md) — long-running eventual consistency
- [Event-Driven Architecture](./event-driven-architecture.md) — communication via events
