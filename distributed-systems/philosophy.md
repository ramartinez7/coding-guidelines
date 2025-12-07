# Philosophy

> Distributed systems are fundamentally about coordinating independent processes that communicate over unreliable networks.

This document explains the *why* behind distributed systems patterns. Unlike local systems where operations are atomic and networks are reliable, distributed systems must embrace uncertainty and design for failure.

---

## 1. The Fundamental Challenges

Building distributed systems means confronting realities that don't exist in single-machine applications.

### The Eight Fallacies of Distributed Computing

Originally identified by L. Peter Deutsch and others at Sun Microsystems:

1. **The network is reliable** — Networks fail, packets get lost, connections drop
2. **Latency is zero** — Every network call has measurable delay (milliseconds to seconds)
3. **Bandwidth is infinite** — Network capacity is limited and must be managed
4. **The network is secure** — Anyone can intercept, modify, or replay messages
5. **Topology doesn't change** — Services move, scale, fail, and redeploy constantly
6. **There is one administrator** — Multiple teams own different parts of the system
7. **Transport cost is zero** — Serialization, bandwidth, and processing all cost resources
8. **The network is homogeneous** — Different protocols, versions, and platforms must interoperate

Every distributed systems pattern exists to address one or more of these fallacies.

---

## 2. The CAP Theorem

**C**onsistency, **A**vailability, **P**artition Tolerance — pick any two (but partition tolerance isn't really optional).

### The Reality

In a distributed system, network partitions *will* occur. When a partition happens, you must choose:

- **CP (Consistency + Partition Tolerance)**: Reject requests to maintain consistency
- **AP (Availability + Partition Tolerance)**: Accept requests knowing data may be stale

```
     Consistent                    Available
         │                             │
         │                             │
    ┌────┴────┐                   ┌────┴────┐
    │ Reject  │                   │ Accept  │
    │ Request │   Partition!      │ Request │
    │ (503)   │◄─────────────────►│ (Stale) │
    └─────────┘                   └─────────┘
```

**There is no perfect solution.** The right choice depends on your business requirements.

### Related Patterns

- [CAP Theorem](./patterns/cap-theorem.md) — detailed explanation with trade-offs
- [Eventual Consistency](./patterns/eventual-consistency.md) — accepting temporary inconsistency

---

## 3. Eventual Consistency

> Strong consistency is expensive. Most systems can tolerate eventual consistency.

### The Insight

In distributed systems, perfect synchronization is expensive and often unnecessary. Instead, accept that:

- Different nodes may temporarily disagree
- Updates propagate asynchronously
- Eventually, all nodes converge to the same state

This isn't weakness—it's pragmatism.

```csharp
// ❌ Synchronous consistency: slow, blocks everything
await _database.UpdateWithLock(item);
await _cache.Invalidate(item.Id);
await _searchIndex.Update(item);
await _eventLog.Append(event);

// ✅ Eventual consistency: fast, resilient
await _database.Update(item);
await _eventBus.Publish(new ItemUpdatedEvent(item.Id));
// Cache, search, and logs update asynchronously
```

### Related Patterns

- [Eventual Consistency](./patterns/eventual-consistency.md)
- [Event Sourcing](./patterns/event-sourcing.md)
- [CQRS](./patterns/cqrs.md)

---

## 4. Coordination Is Expensive

Every time systems must agree on something (leader election, distributed locks, consensus), performance suffers.

### Minimize Coordination

| Pattern | Coordination Cost | Use When |
|---------|------------------|----------|
| **No coordination** | None | Independent operations |
| **Optimistic concurrency** | Low | Conflicts are rare |
| **Two-phase commit** | High | Strong consistency required |
| **Consensus** | Very high | Critical decisions (Raft, Paxos) |

Design to reduce coordination:

```csharp
// ❌ High coordination: every request needs a lock
lock (_globalCounter)
{
    _counter++;
    await ProcessRequest(request);
}

// ✅ Low coordination: local counters, merge later
var localCounter = _counterPerNode[nodeId];
localCounter.Increment();
await ProcessRequest(request);
// Periodically reconcile counters across nodes
```

### Related Patterns

- [Two-Phase Commit](./patterns/two-phase-commit.md) — coordination with strong consistency
- [Saga Pattern](./patterns/saga-pattern.md) — coordination with eventual consistency

---

## 5. Embrace Asynchrony

Synchronous calls in distributed systems create coupling, cascading failures, and timeouts.

### Async by Default

```
     Synchronous                       Asynchronous
     
 ┌─────────┐                       ┌─────────┐
 │Service A│                       │Service A│
 └────┬────┘                       └────┬────┘
      │                                 │
      │ HTTP POST                       │ Publish Event
      │ (blocks)                        │ (fire & forget)
      ▼                                 ▼
 ┌─────────┐                       ┌──────────┐
 │Service B│                       │Event Bus │
 └─────────┘                       └────┬─────┘
      │                                 │
      │ (A waits...)                    │ Subscribe
      │                                 ▼
      │                            ┌─────────┐
      │                            │Service B│
      │                            └─────────┘
      │
 If B is down,                    If B is down,
 A fails too!                     A continues!
```

### Related Patterns

- [Event-Driven Architecture](./patterns/event-driven-architecture.md)
- [Saga Pattern](./patterns/saga-pattern.md)

---

## 6. Design for Failure

In distributed systems, failure is not an exception—it's the norm.

### Failure Modes

| Component | Failure | Pattern |
|-----------|---------|---------|
| **Service** | Crashes | Circuit breaker, retry with backoff |
| **Network** | Partition | Graceful degradation |
| **Database** | Unavailable | Read replicas, cache |
| **Message** | Lost | Idempotency, acknowledgments |
| **Clock** | Skew | Logical clocks, vector clocks |

```csharp
// ❌ Assumes success
var user = await _userService.GetUser(id);
return user.Name;

// ✅ Expects failure
var result = await _userService.GetUser(id);
return result.Match(
    onSuccess: user => user.Name,
    onFailure: error => error switch
    {
        ServiceUnavailable => "Service temporarily down",
        NotFound => "User not found",
        Timeout => "Request timed out",
        _ => "Unknown error"
    });
```

### Related Patterns

- [Circuit Breaker](./patterns/circuit-breaker.md)
- [Saga Pattern](./patterns/saga-pattern.md) — handling failures in distributed transactions

---

## 7. Idempotency Is Non-Negotiable

Networks retry. If your operations aren't idempotent, retries cause corruption.

> An idempotent operation produces the same result whether executed once or many times.

```csharp
// ❌ Not idempotent: retry doubles the charge
public void ChargeCustomer(CustomerId id, decimal amount)
{
    _balance[id] -= amount;
}

// ✅ Idempotent: retry is safe
public void ChargeCustomer(IdempotencyKey key, CustomerId id, decimal amount)
{
    if (_processedKeys.Contains(key))
        return;  // Already processed
    
    _balance[id] -= amount;
    _processedKeys.Add(key);
}
```

### Related Patterns

- [Event-Driven Architecture](./patterns/event-driven-architecture.md) — idempotent event processing
- [Saga Pattern](./patterns/saga-pattern.md) — idempotent compensation actions

---

## 8. Observability Over Debugging

You can't attach a debugger to a distributed system. You must design for observability from the start.

### The Three Pillars

| Pillar | Purpose | Example |
|--------|---------|---------|
| **Logs** | What happened? | "Order 123 created by user 456" |
| **Metrics** | How much/often? | "99th percentile latency: 250ms" |
| **Traces** | Where did time go? | "Request spent 150ms in database" |

```csharp
// Every operation carries context
public async Task ProcessOrder(
    CorrelationId correlationId,
    Order order)
{
    _logger.LogInformation(
        "Processing order {OrderId} [Correlation: {CorrelationId}]",
        order.Id,
        correlationId);
    
    using var activity = _activitySource.StartActivity("ProcessOrder");
    activity?.SetTag("order.id", order.Id);
    
    _metrics.RecordOrderProcessed();
    
    await _repository.SaveAsync(order);
}
```

### Related Patterns

- [Event-Driven Architecture](./patterns/event-driven-architecture.md) — observing event flows
- [Circuit Breaker](./patterns/circuit-breaker.md) — monitoring service health

---

## 9. Time Is Relative

Clocks on different machines drift. You cannot rely on wall-clock time for ordering.

```csharp
// ❌ Wall-clock time: machines disagree
var timestamp = DateTime.UtcNow;
if (event1.Timestamp < event2.Timestamp)
    // Which actually happened first? Unknown!

// ✅ Logical clocks: explicit ordering
var event1 = new Event(content, lamportClock: 5);
var event2 = new Event(content, lamportClock: 7);
if (event1.Clock < event2.Clock)
    // Definitely happened before
```

### Related Patterns

- [Event Sourcing](./patterns/event-sourcing.md) — events capture causal ordering
- [Eventual Consistency](./patterns/eventual-consistency.md) — handling concurrent updates

---

## 10. Data Gravity

Data has mass. Moving it is expensive. Compute should move to data, not vice versa.

```
     Bad: Move Data to Compute          Good: Move Compute to Data
     
     ┌──────────────┐                   ┌──────────────┐
     │  Service A   │                   │  Service A   │
     │   (Canada)   │                   │   (Canada)   │
     └──────┬───────┘                   └──────┬───────┘
            │                                  │
            │ Fetch 1GB                        │ Send query
            │ (slow)                           │ (fast)
            ▼                                  ▼
     ┌──────────────┐                   ┌──────────────┐
     │  Database    │                   │  Database    │
     │  (Australia) │                   │  (Australia) │
     └──────────────┘                   │  Executes    │
                                        │  Sends 10KB  │
                                        └──────────────┘
```

### Related Patterns

- [CQRS](./patterns/cqrs.md) — optimizing read models for locality
- [Event Sourcing](./patterns/event-sourcing.md) — replaying events locally

---

## Summary

These ten principles guide distributed systems design:

| Principle | Key Question |
|-----------|--------------|
| **Fundamental Challenges** | Am I assuming the network is reliable? |
| **CAP Theorem** | What happens during a partition? |
| **Eventual Consistency** | Can I tolerate temporary inconsistency? |
| **Coordination Is Expensive** | How can I reduce synchronization? |
| **Embrace Asynchrony** | Can this be fire-and-forget? |
| **Design for Failure** | What happens when this fails? |
| **Idempotency** | Is this safe to retry? |
| **Observability** | Can I debug this in production? |
| **Time Is Relative** | Am I relying on clock synchronization? |
| **Data Gravity** | Should I move data or computation? |

When designing distributed systems, ask these questions. The patterns in this catalog are tools to answer "yes" to each one.
