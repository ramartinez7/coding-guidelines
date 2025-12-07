# CAP Theorem

> Consistency, Availability, Partition Tolerance—in a distributed system experiencing a network partition, you must choose between consistency and availability.

## Problem

Distributed systems must handle network partitions (when nodes cannot communicate). During a partition, it's impossible to provide both perfect consistency and perfect availability. You must choose which property to sacrifice.

## The CAP Triangle

```
               Consistency
                    /\
                   /  \
                  /    \
                 /  CP  \
                /        \
               /          \
              /            \
             /______________\
        Partition      AP      Availability
        Tolerance
```

### The Three Properties

**Consistency (C)**: All nodes see the same data at the same time

**Availability (A)**: Every request receives a response (success or failure)

**Partition Tolerance (P)**: System continues operating despite network failures

## The Reality: Partition Tolerance Is Required

Network partitions *will* happen in distributed systems. Therefore, you don't really choose from three options—you choose between:

- **CP (Consistency over Availability)**: Reject requests to maintain consistency
- **AP (Availability over Consistency)**: Accept requests with potentially stale data

## Example

### ❌ Ignoring Partitions

```csharp
public class DistributedCache
{
    private readonly List<CacheNode> _nodes;
    
    public async Task<string> Get(string key)
    {
        // Assumes all nodes are always reachable
        var tasks = _nodes.Select(node => node.GetAsync(key));
        var results = await Task.WhenAll(tasks);
        
        // What if nodes disagree? What if some are unreachable?
        return results.First(r => r != null);
    }
}
```

### ✅ CP System: Consistency First

```csharp
/// <summary>
/// CP system: Rejects requests during partition to maintain consistency.
/// Example: Banking systems, inventory management.
/// </summary>
public class ConsistentCache
{
    private readonly List<CacheNode> _nodes;
    private readonly int _requiredQuorum;
    
    public ConsistentCache(List<CacheNode> nodes)
    {
        _nodes = nodes;
        _requiredQuorum = (nodes.Count / 2) + 1;  // Majority
    }
    
    public async Task<Result<string, CacheError>> Get(string key)
    {
        var reachableNodes = await GetReachableNodes();
        
        // Require quorum for consistency
        if (reachableNodes.Count < _requiredQuorum)
        {
            return Result<string, CacheError>.Failure(
                new CacheError.InsufficientNodes(
                    $"Cannot guarantee consistency: only {reachableNodes.Count}/{_nodes.Count} nodes reachable"));
        }
        
        var tasks = reachableNodes.Select(node => node.GetAsync(key));
        var results = await Task.WhenAll(tasks);
        
        // All reachable nodes must agree
        var distinctValues = results.Where(r => r != null).Distinct().ToList();
        
        if (distinctValues.Count > 1)
        {
            return Result<string, CacheError>.Failure(
                new CacheError.InconsistentState("Nodes have conflicting values"));
        }
        
        return Result<string, CacheError>.Success(distinctValues.FirstOrDefault());
    }
    
    public async Task<Result<Unit, CacheError>> Set(string key, string value)
    {
        var reachableNodes = await GetReachableNodes();
        
        // Require quorum for consistency
        if (reachableNodes.Count < _requiredQuorum)
        {
            // Reject write to maintain consistency
            return Result<Unit, CacheError>.Failure(
                new CacheError.InsufficientNodes(
                    $"Cannot guarantee consistency during write"));
        }
        
        // Write to all reachable nodes
        var tasks = reachableNodes.Select(node => node.SetAsync(key, value));
        await Task.WhenAll(tasks);
        
        return Result<Unit, CacheError>.Success(Unit.Value);
    }
    
    private async Task<List<CacheNode>> GetReachableNodes()
    {
        var healthChecks = await Task.WhenAll(
            _nodes.Select(async node => new
            {
                Node = node,
                IsReachable = await node.IsReachableAsync()
            }));
        
        return healthChecks
            .Where(check => check.IsReachable)
            .Select(check => check.Node)
            .ToList();
    }
}

public abstract record CacheError
{
    public sealed record InsufficientNodes(string Message) : CacheError;
    public sealed record InconsistentState(string Message) : CacheError;
    public sealed record Timeout(string Message) : CacheError;
}
```

### ✅ AP System: Availability First

```csharp
/// <summary>
/// AP system: Accepts requests during partition, allowing stale reads.
/// Example: Social media feeds, product catalogs, metrics dashboards.
/// </summary>
public class AvailableCache
{
    private readonly List<CacheNode> _nodes;
    
    public async Task<Result<string, CacheError>> Get(string key)
    {
        // Try all nodes, return first success
        foreach (var node in _nodes)
        {
            try
            {
                var result = await node.GetAsync(key);
                if (result != null)
                {
                    // Data might be stale, but we're available!
                    return Result<string, CacheError>.Success(result);
                }
            }
            catch
            {
                // Node unreachable, try next
                continue;
            }
        }
        
        return Result<string, CacheError>.Failure(
            new CacheError.AllNodesUnreachable("No nodes available"));
    }
    
    public async Task<Result<Unit, CacheError>> Set(string key, string value)
    {
        var successCount = 0;
        
        // Best-effort write to all nodes
        foreach (var node in _nodes)
        {
            try
            {
                await node.SetAsync(key, value);
                successCount++;
            }
            catch
            {
                // Node unreachable, continue anyway
                continue;
            }
        }
        
        // Accept write if ANY node succeeded
        if (successCount > 0)
        {
            // Nodes may be inconsistent, but we're available!
            return Result<Unit, CacheError>.Success(Unit.Value);
        }
        
        return Result<Unit, CacheError>.Failure(
            new CacheError.AllNodesUnreachable("No nodes available for write"));
    }
}
```

## Decision Guide

### Choose CP When:

- **Financial transactions**: Cannot tolerate inconsistent balances
- **Inventory systems**: Cannot oversell products
- **Access control**: Cannot grant unauthorized access
- **Strong business invariants**: Cannot violate domain rules

**Example:** Banking system must reject transfers if it cannot guarantee both accounts update atomically.

### Choose AP When:

- **Social media feeds**: Stale data is acceptable
- **Product catalogs**: Eventual consistency is fine
- **Metrics/dashboards**: Approximate data is useful
- **User-generated content**: Availability matters more than instant consistency

**Example:** Twitter feed can show slightly stale data during network issues—better than showing nothing.

## Tunable Consistency

Modern systems often allow tuning the consistency level:

```csharp
public enum ConsistencyLevel
{
    /// <summary>
    /// Strong consistency - read latest write (CP behavior)
    /// </summary>
    Strong,
    
    /// <summary>
    /// Read from quorum - balance consistency and availability
    /// </summary>
    Quorum,
    
    /// <summary>
    /// Read from any node - favor availability (AP behavior)
    /// </summary>
    Eventual
}

public class TunableCache
{
    public async Task<Result<string, CacheError>> Get(
        string key,
        ConsistencyLevel level = ConsistencyLevel.Quorum)
    {
        return level switch
        {
            ConsistencyLevel.Strong => await GetStrong(key),
            ConsistencyLevel.Quorum => await GetQuorum(key),
            ConsistencyLevel.Eventual => await GetEventual(key),
            _ => throw new ArgumentException($"Unknown consistency level: {level}")
        };
    }
    
    private async Task<Result<string, CacheError>> GetStrong(string key)
    {
        // Read from ALL nodes, require agreement
        var tasks = _nodes.Select(node => node.GetAsync(key));
        var results = await Task.WhenAll(tasks);
        
        var distinctValues = results.Distinct().ToList();
        if (distinctValues.Count > 1)
        {
            return Result<string, CacheError>.Failure(
                new CacheError.InconsistentState("Nodes disagree"));
        }
        
        return Result<string, CacheError>.Success(distinctValues.First());
    }
    
    private async Task<Result<string, CacheError>> GetQuorum(string key)
    {
        // Read from majority, require agreement
        var quorum = (_nodes.Count / 2) + 1;
        var reachableNodes = await GetReachableNodes();
        
        if (reachableNodes.Count < quorum)
        {
            return Result<string, CacheError>.Failure(
                new CacheError.InsufficientNodes("Quorum not available"));
        }
        
        var tasks = reachableNodes.Take(quorum).Select(node => node.GetAsync(key));
        var results = await Task.WhenAll(tasks);
        
        return Result<string, CacheError>.Success(results.First());
    }
    
    private async Task<Result<string, CacheError>> GetEventual(string key)
    {
        // Read from ANY node
        foreach (var node in _nodes)
        {
            try
            {
                var result = await node.GetAsync(key);
                if (result != null)
                {
                    return Result<string, CacheError>.Success(result);
                }
            }
            catch
            {
                continue;
            }
        }
        
        return Result<string, CacheError>.Failure(
            new CacheError.AllNodesUnreachable("No nodes available"));
    }
}
```

## Why It's a Fundamental Trade-off

1. **Network partitions are inevitable**: Cables break, switches fail, data centers lose connectivity
2. **You cannot have perfect consistency and availability**: During a partition, responding with stale data (AP) means sacrificing consistency, while rejecting requests (CP) means sacrificing availability
3. **The choice depends on business requirements**: There is no universally correct answer

## Symptoms

- System becomes unavailable during network issues (might be choosing CP inappropriately)
- Data inconsistencies appear across nodes (might be choosing AP inappropriately)
- Unclear behavior during network partitions
- Assuming networks never fail

## Benefits

- **Explicit trade-offs**: Forces conscious decision about consistency vs. availability
- **Appropriate design**: Match system behavior to business requirements
- **Predictable failures**: Know how system behaves during partitions
- **Type-safe configurations**: Use enums/types to make consistency level explicit

## See Also

- [Eventual Consistency](./eventual-consistency.md) — accepting temporary inconsistency
- [Two-Phase Commit](./two-phase-commit.md) — strong consistency across nodes
- [Saga Pattern](./saga-pattern.md) — eventual consistency for long-running transactions
- [Quorum Reads and Writes](./quorum-reads-writes.md) — tunable consistency
