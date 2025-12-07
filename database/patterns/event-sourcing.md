# Event Sourcing

> Store domain events as an immutable log—derive current state through projection and enable time-travel queries.

## Problem

Traditional CRUD systems only store current state. Historical data is lost, debugging is difficult, and it's impossible to reconstruct how the system arrived at the current state or replay events for testing/migration.

## Example

### ❌ Before (State-Based CRUD)

```sql
CREATE TABLE dbo.BankAccount (
    AccountId INT NOT NULL,
    Balance DECIMAL(18,2) NOT NULL,
    ModifiedAt DATETIME2(0) NOT NULL,
    
    CONSTRAINT PK_BankAccount PRIMARY KEY (AccountId)
);

-- Transaction history
UPDATE dbo.BankAccount SET Balance = Balance - 100 WHERE AccountId = 123;
UPDATE dbo.BankAccount SET Balance = Balance + 50 WHERE AccountId = 123;
UPDATE dbo.BankAccount SET Balance = Balance - 25 WHERE AccountId = 123;

-- Current balance is available, but:
-- - Cannot reconstruct how we got here
-- - Cannot audit individual transactions
-- - Cannot replay events
-- - Cannot answer "what was balance at 2pm yesterday?"
```

### ✅ After (Event Sourcing)

```sql
-- Event store: append-only log of everything that happened
CREATE TABLE dbo.BankAccountEvent (
    EventId BIGINT IDENTITY(1,1) NOT NULL,
    AccountId INT NOT NULL,
    EventType VARCHAR(50) NOT NULL,
    Amount DECIMAL(18,2) NULL,
    EventData NVARCHAR(MAX) NOT NULL,  -- JSON with full event details
    CreatedAt DATETIME2(3) NOT NULL DEFAULT SYSDATETIME(),
    CreatedBy INT NOT NULL,
    
    CONSTRAINT PK_BankAccountEvent PRIMARY KEY (EventId),
    CONSTRAINT CK_BankAccountEvent_EventType CHECK (
        EventType IN ('AccountOpened', 'MoneyDeposited', 'MoneyWithdrawn', 'AccountClosed')
    )
);

-- Create account
INSERT INTO dbo.BankAccountEvent (AccountId, EventType, Amount, EventData, CreatedBy)
VALUES (123, 'AccountOpened', 0, '{"initialBalance": 0}', @UserId);

-- Deposit money
INSERT INTO dbo.BankAccountEvent (AccountId, EventType, Amount, EventData, CreatedBy)
VALUES (123, 'MoneyDeposited', 100, '{"source": "ATM", "location": "NYC"}', @UserId);

-- Withdraw money
INSERT INTO dbo.BankAccountEvent (AccountId, EventType, Amount, EventData, CreatedBy)
VALUES (123, 'MoneyWithdrawn', 50, '{"destination": "Check", "checkNumber": "1001"}', @UserId);

-- Current state: calculated from events
SELECT 
    AccountId,
    SUM(CASE 
        WHEN EventType = 'MoneyDeposited' THEN Amount
        WHEN EventType = 'MoneyWithdrawn' THEN -Amount
        ELSE 0
    END) AS Balance
FROM dbo.BankAccountEvent
WHERE AccountId = 123
GROUP BY AccountId;

-- Historical state: balance at any point in time
SELECT 
    AccountId,
    SUM(CASE 
        WHEN EventType = 'MoneyDeposited' THEN Amount
        WHEN EventType = 'MoneyWithdrawn' THEN -Amount
        ELSE 0
    END) AS Balance
FROM dbo.BankAccountEvent
WHERE AccountId = 123
  AND CreatedAt <= '2024-01-15T14:00:00'  -- Balance at 2pm on Jan 15
GROUP BY AccountId;
```

## Event Store Design

### Core Event Table

```sql
CREATE TABLE dbo.Event (
    EventId BIGINT IDENTITY(1,1) NOT NULL,
    AggregateId UNIQUEIDENTIFIER NOT NULL,      -- Entity this event belongs to
    AggregateType VARCHAR(100) NOT NULL,        -- Order, Customer, etc.
    EventType VARCHAR(100) NOT NULL,            -- OrderCreated, OrderShipped, etc.
    EventVersion INT NOT NULL,                  -- Event schema version
    EventData NVARCHAR(MAX) NOT NULL,           -- JSON event payload
    Metadata NVARCHAR(MAX) NULL,                -- Correlation ID, causation, etc.
    CreatedAt DATETIME2(3) NOT NULL DEFAULT SYSDATETIME(),
    CreatedBy INT NOT NULL,
    
    CONSTRAINT PK_Event PRIMARY KEY (EventId),
    CONSTRAINT CK_Event_EventData_Json CHECK (ISJSON(EventData) = 1)
);

-- Index for aggregate queries
CREATE NONCLUSTERED INDEX IX_Event_AggregateId
ON dbo.Event(AggregateId, EventId);

-- Index for event type queries
CREATE NONCLUSTERED INDEX IX_Event_Type
ON dbo.Event(AggregateType, EventType, EventId);
```

### Projection (Read Model)

```sql
-- Materialized view of current state
CREATE TABLE dbo.OrderProjection (
    OrderId UNIQUEIDENTIFIER NOT NULL,
    CustomerId INT NOT NULL,
    Status VARCHAR(20) NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    CreatedAt DATETIME2(0) NOT NULL,
    LastEventId BIGINT NOT NULL,  -- Track which events we've processed
    
    CONSTRAINT PK_OrderProjection PRIMARY KEY (OrderId)
);

-- Rebuild projection from events
TRUNCATE TABLE dbo.OrderProjection;

INSERT INTO dbo.OrderProjection (OrderId, CustomerId, Status, Total, CreatedAt, LastEventId)
SELECT 
    e.AggregateId AS OrderId,
    MAX(CAST(JSON_VALUE(e.EventData, '$.customerId') AS INT)) AS CustomerId,
    MAX(CASE 
        WHEN e.EventType = 'OrderCreated' THEN 'Pending'
        WHEN e.EventType = 'OrderShipped' THEN 'Shipped'
        WHEN e.EventType = 'OrderDelivered' THEN 'Delivered'
        ELSE 'Unknown'
    END) AS Status,
    SUM(CAST(JSON_VALUE(e.EventData, '$.amount') AS DECIMAL(18,2))) AS Total,
    MIN(e.CreatedAt) AS CreatedAt,
    MAX(e.EventId) AS LastEventId
FROM dbo.Event e
WHERE e.AggregateType = 'Order'
GROUP BY e.AggregateId;
```

## Event Processor

Process events to update projections:

```sql
CREATE PROCEDURE dbo.usp_ProcessOrderEvents
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Get unprocessed events
    DECLARE @LastProcessedEventId BIGINT;
    
    SELECT @LastProcessedEventId = ISNULL(MAX(LastEventId), 0)
    FROM dbo.OrderProjection;
    
    -- Process new events
    DECLARE @Events TABLE (
        EventId BIGINT,
        AggregateId UNIQUEIDENTIFIER,
        EventType VARCHAR(100),
        EventData NVARCHAR(MAX),
        CreatedAt DATETIME2(3)
    );
    
    INSERT INTO @Events
    SELECT 
        EventId,
        AggregateId,
        EventType,
        EventData,
        CreatedAt
    FROM dbo.Event
    WHERE AggregateType = 'Order'
      AND EventId > @LastProcessedEventId
    ORDER BY EventId;
    
    -- Update projections based on events
    MERGE dbo.OrderProjection AS target
    USING (
        SELECT 
            AggregateId,
            MAX(EventId) AS LastEventId,
            MAX(CreatedAt) AS LastModified
        FROM @Events
        GROUP BY AggregateId
    ) AS source
    ON target.OrderId = source.AggregateId
    WHEN MATCHED THEN
        UPDATE SET 
            LastEventId = source.LastEventId
    WHEN NOT MATCHED THEN
        INSERT (OrderId, CustomerId, Status, Total, CreatedAt, LastEventId)
        VALUES (
            source.AggregateId,
            0,  -- Extract from events
            'Pending',
            0,
            source.LastModified,
            source.LastEventId
        );
END;
```

## Snapshotting

Optimize by creating periodic snapshots:

```sql
CREATE TABLE dbo.OrderSnapshot (
    OrderId UNIQUEIDENTIFIER NOT NULL,
    SnapshotData NVARCHAR(MAX) NOT NULL,  -- Full order state as JSON
    SnapshotVersion BIGINT NOT NULL,       -- EventId when snapshot taken
    CreatedAt DATETIME2(3) NOT NULL,
    
    CONSTRAINT PK_OrderSnapshot PRIMARY KEY (OrderId, SnapshotVersion)
);

-- Rebuild state: snapshot + events after snapshot
SELECT 
    -- Start with snapshot
    s.SnapshotData,
    s.SnapshotVersion
FROM dbo.OrderSnapshot s
WHERE s.OrderId = @OrderId
  AND s.SnapshotVersion = (
      SELECT MAX(SnapshotVersion) 
      FROM dbo.OrderSnapshot 
      WHERE OrderId = @OrderId
  )

UNION ALL

-- Apply events after snapshot
SELECT 
    e.EventData,
    e.EventId
FROM dbo.Event e
WHERE e.AggregateId = @OrderId
  AND e.EventId > (
      SELECT MAX(SnapshotVersion) 
      FROM dbo.OrderSnapshot 
      WHERE OrderId = @OrderId
  )
ORDER BY SnapshotVersion;
```

## Event Types

### Command Events (What Happened)

```sql
-- Domain events that describe what happened
INSERT INTO dbo.Event (AggregateId, AggregateType, EventType, EventData, CreatedBy)
VALUES (
    @OrderId,
    'Order',
    'OrderCreated',
    JSON_OBJECT(
        'customerId': @CustomerId,
        'total': @Total,
        'items': JSON_ARRAY(...)
    ),
    @UserId
);
```

### Integration Events (For External Systems)

```sql
-- Events published to message bus for other services
CREATE TABLE dbo.IntegrationEvent (
    IntegrationEventId BIGINT IDENTITY(1,1) NOT NULL,
    EventId BIGINT NOT NULL,  -- Links to Event table
    EventType VARCHAR(100) NOT NULL,
    EventData NVARCHAR(MAX) NOT NULL,
    PublishedAt DATETIME2(3) NULL,
    PublishCount INT NOT NULL DEFAULT 0,
    
    CONSTRAINT PK_IntegrationEvent PRIMARY KEY (IntegrationEventId),
    CONSTRAINT FK_IntegrationEvent_Event FOREIGN KEY (EventId)
        REFERENCES dbo.Event(EventId)
);

-- Mark as published
UPDATE dbo.IntegrationEvent
SET 
    PublishedAt = SYSDATETIME(),
    PublishCount = PublishCount + 1
WHERE IntegrationEventId = @EventId;
```

## Querying Events

### All Events for Aggregate

```sql
-- Complete history of an order
SELECT 
    EventId,
    EventType,
    EventData,
    CreatedAt,
    CreatedBy
FROM dbo.Event
WHERE AggregateId = @OrderId
ORDER BY EventId;
```

### Events by Type

```sql
-- All order shipment events
SELECT 
    e.EventId,
    e.AggregateId AS OrderId,
    JSON_VALUE(e.EventData, '$.trackingNumber') AS TrackingNumber,
    e.CreatedAt AS ShippedAt
FROM dbo.Event e
WHERE e.AggregateType = 'Order'
  AND e.EventType = 'OrderShipped'
ORDER BY e.CreatedAt DESC;
```

### Time-Travel Queries

```sql
-- State at specific point in time
SELECT 
    AggregateId AS OrderId,
    COUNT(*) AS EventCount,
    MAX(CASE WHEN EventType = 'OrderShipped' THEN 1 ELSE 0 END) AS WasShipped
FROM dbo.Event
WHERE AggregateType = 'Order'
  AND CreatedAt <= '2024-01-15T14:00:00'
GROUP BY AggregateId;
```

## Why State-Based CRUD Is a Problem

1. **Lost history**: Cannot see how state changed over time
2. **No audit trail**: Cannot determine who did what
3. **Debugging difficulty**: Cannot reproduce bugs by replaying events
4. **Concurrency issues**: Lost updates in state-based systems
5. **Migration complexity**: Cannot test migrations on historical data

## Symptoms

- "We need an audit log" features added later
- Unable to answer "what changed?" questions
- Complex triggers trying to track history
- Multiple history tables duplicating effort

## Benefits

- **Complete audit trail**: Every change recorded forever
- **Time travel**: Query state at any point in history
- **Debugging**: Replay events to reproduce bugs
- **Testing**: Replay production events in test environment
- **Analytics**: Understand how system behavior changes over time
- **Event-driven architecture**: Natural fit for microservices

## Trade-offs

- **Complexity**: More complex than simple CRUD
- **Storage**: Event store grows indefinitely (mitigate with snapshots)
- **Query performance**: Projections needed for efficient reads
- **Learning curve**: Different mental model than CRUD
- **Eventual consistency**: Projections may lag behind events

## When to Use

✅ Use event sourcing when:
- Audit trail is critical (financial, healthcare)
- Need to replay/reprocess historical data
- Building event-driven architecture
- Complex domain with many state transitions

❌ Don't use when:
- Simple CRUD application
- No audit requirements
- Read-heavy with rare writes
- Team unfamiliar with pattern

## See Also

- [Temporal Tables](./temporal-tables.md)
- [Audit Columns](./audit-columns.md)
- [Idempotent Operations](./idempotent-operations.md)
- [Soft Deletes](./soft-deletes.md)
