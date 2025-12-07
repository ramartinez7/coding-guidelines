# Pessimistic Locking

> Lock rows during read to prevent concurrent modifications—guarantee exclusive access during updates.

## Problem

In high-conflict scenarios where multiple users frequently modify the same records, optimistic concurrency leads to excessive retries and poor user experience. Users need guaranteed exclusive access.

## Example

### ❌ Before (Race Condition with Read-Then-Update)

```sql
-- User A reads inventory
SELECT ProductId, StockQuantity
FROM dbo.Product
WHERE ProductId = 123;
-- StockQuantity = 10

-- User B reads same inventory (same time)
SELECT ProductId, StockQuantity
FROM dbo.Product
WHERE ProductId = 123;
-- StockQuantity = 10

-- User A places order for 8 units
UPDATE dbo.Product
SET StockQuantity = StockQuantity - 8
WHERE ProductId = 123;
-- StockQuantity now 2

-- User B places order for 5 units (oversell!)
UPDATE dbo.Product
SET StockQuantity = StockQuantity - 5
WHERE ProductId = 123;
-- ❌ StockQuantity now -3 (negative stock!)
```

### ✅ After (Pessimistic Lock)

```sql
-- User A locks the row during read
BEGIN TRANSACTION;

SELECT ProductId, StockQuantity
FROM dbo.Product WITH (UPDLOCK, HOLDLOCK)
WHERE ProductId = 123;
-- StockQuantity = 10, row is locked

-- User B tries to read same row
-- ⏳ BLOCKED until User A commits

-- User A validates and updates
IF @StockQuantity >= 8
BEGIN
    UPDATE dbo.Product
    SET StockQuantity = StockQuantity - 8
    WHERE ProductId = 123;
END;

COMMIT TRANSACTION;
-- ✅ User B can now proceed with latest stock value
```

## Lock Hints

### UPDLOCK

Acquire update lock instead of shared lock—signals intent to modify:

```sql
-- Without UPDLOCK: Shared lock → potential deadlock
SELECT * FROM dbo.Product WHERE ProductId = 123;
UPDATE dbo.Product SET Price = 99.99 WHERE ProductId = 123;

-- With UPDLOCK: Update lock → prevents deadlock
SELECT * FROM dbo.Product WITH (UPDLOCK)
WHERE ProductId = 123;
UPDATE dbo.Product SET Price = 99.99 WHERE ProductId = 123;
```

### HOLDLOCK (or SERIALIZABLE)

Hold locks until transaction ends:

```sql
-- Without HOLDLOCK: Lock released after SELECT
BEGIN TRANSACTION;
SELECT * FROM dbo.Product WHERE ProductId = 123;
-- Lock released here—other transactions can modify
UPDATE dbo.Product SET Price = 99.99 WHERE ProductId = 123;
COMMIT;

-- With HOLDLOCK: Lock held until COMMIT
BEGIN TRANSACTION;
SELECT * FROM dbo.Product WITH (HOLDLOCK)
WHERE ProductId = 123;
-- Lock held—no modifications allowed
UPDATE dbo.Product SET Price = 99.99 WHERE ProductId = 123;
COMMIT;
```

### UPDLOCK + HOLDLOCK Combined

Best practice for pessimistic locking:

```sql
CREATE PROCEDURE dbo.usp_ReserveInventory
    @ProductId INT,
    @Quantity INT
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Lock row for update
        DECLARE @StockQuantity INT;
        
        SELECT @StockQuantity = StockQuantity
        FROM dbo.Product WITH (UPDLOCK, HOLDLOCK)
        WHERE ProductId = @ProductId;
        
        -- Validate availability
        IF @StockQuantity IS NULL
        BEGIN
            THROW 50001, 'Product not found', 1;
        END;
        
        IF @StockQuantity < @Quantity
        BEGIN
            THROW 50002, 'Insufficient stock', 1;
        END;
        
        -- Reserve inventory
        UPDATE dbo.Product
        SET StockQuantity = StockQuantity - @Quantity
        WHERE ProductId = @ProductId;
        
        COMMIT TRANSACTION;
        
        SELECT 'SUCCESS' AS Result, @Quantity AS ReservedQuantity;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        THROW;
    END CATCH;
END;
```

## ROWLOCK Hint

Lock at row level instead of page or table:

```sql
-- Lock only specific rows, not entire page
SELECT *
FROM dbo.Product WITH (UPDLOCK, HOLDLOCK, ROWLOCK)
WHERE CategoryId = 5;

-- Without ROWLOCK: Might lock entire page
-- With ROWLOCK: Locks only matching rows
```

## XLOCK (Exclusive Lock)

Acquire exclusive lock immediately—block all readers:

```sql
-- Block all access (reads and writes)
SELECT *
FROM dbo.Product WITH (XLOCK, HOLDLOCK)
WHERE ProductId = 123;

-- Use for critical sections where no access allowed
BEGIN TRANSACTION;

SELECT @NextOrderId = ISNULL(MAX(OrderId), 0) + 1
FROM dbo.Order WITH (XLOCK, TABLOCKX);

INSERT INTO dbo.Order (OrderId, CustomerId, Total)
VALUES (@NextOrderId, @CustomerId, @Total);

COMMIT;
```

## TABLOCKX

Lock entire table exclusively:

```sql
-- Lock entire table for bulk operation
BEGIN TRANSACTION;

-- Prevent all access to table
SELECT TOP 0 * FROM dbo.Archive WITH (TABLOCKX);

-- Bulk insert
INSERT INTO dbo.Archive
SELECT * FROM dbo.ActiveOrders
WHERE OrderDate < DATEADD(YEAR, -1, SYSDATETIME());

-- Delete from source
DELETE FROM dbo.ActiveOrders
WHERE OrderDate < DATEADD(YEAR, -1, SYSDATETIME());

COMMIT;
```

## Timeout Configuration

Set lock timeout to avoid indefinite blocking:

```sql
-- Set 5 second timeout
SET LOCK_TIMEOUT 5000;

BEGIN TRY
    BEGIN TRANSACTION;
    
    SELECT * FROM dbo.Product WITH (UPDLOCK, HOLDLOCK)
    WHERE ProductId = 123;
    
    UPDATE dbo.Product SET Price = 99.99 WHERE ProductId = 123;
    
    COMMIT;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK;
    
    IF ERROR_NUMBER() = 1222  -- Lock timeout
    BEGIN
        RAISERROR('Resource is locked, please retry', 16, 1);
    END
    ELSE
    BEGIN
        THROW;
    END;
END CATCH;

-- Reset to no timeout
SET LOCK_TIMEOUT -1;
```

## Application-Level Locks

Use named locks for custom coordination:

```sql
-- Acquire application lock
DECLARE @Result INT;

EXEC @Result = sp_getapplock 
    @Resource = 'DailyReportGeneration',
    @LockMode = 'Exclusive',
    @LockOwner = 'Transaction',
    @LockTimeout = 10000;

IF @Result < 0
BEGIN
    RAISERROR('Could not acquire lock', 16, 1);
    RETURN;
END;

BEGIN TRY
    -- Generate report (only one process can run this)
    EXEC dbo.usp_GenerateDailyReport;
    
    -- Release lock
    EXEC sp_releaseapplock 
        @Resource = 'DailyReportGeneration',
        @LockOwner = 'Transaction';
END TRY
BEGIN CATCH
    EXEC sp_releaseapplock 
        @Resource = 'DailyReportGeneration',
        @LockOwner = 'Transaction';
    
    THROW;
END CATCH;
```

## Pessimistic vs Optimistic Comparison

| Aspect | Pessimistic | Optimistic |
|--------|-------------|-----------|
| **When to use** | High conflict, must succeed first try | Low conflict, retries acceptable |
| **Locking** | Lock on read | No locks, check on write |
| **Concurrency** | Low (blocks other users) | High (no blocking) |
| **Retries** | Not needed | May need retries |
| **Performance** | Slower for reads | Slower on conflicts |
| **Use cases** | Inventory, seat reservations | Form edits, settings |

## Inventory Management Example

```sql
CREATE PROCEDURE dbo.usp_PlaceOrder
    @CustomerId INT,
    @Items dbo.OrderItemType READONLY
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Lock all products being ordered
        DECLARE @Products TABLE (
            ProductId INT,
            StockQuantity INT
        );
        
        INSERT INTO @Products (ProductId, StockQuantity)
        SELECT p.ProductId, p.StockQuantity
        FROM dbo.Product p WITH (UPDLOCK, HOLDLOCK, ROWLOCK)
        INNER JOIN @Items i ON p.ProductId = i.ProductId;
        
        -- Validate stock for all items
        IF EXISTS (
            SELECT 1
            FROM @Products p
            INNER JOIN @Items i ON p.ProductId = i.ProductId
            WHERE p.StockQuantity < i.Quantity
        )
        BEGIN
            THROW 50001, 'Insufficient stock for one or more items', 1;
        END;
        
        -- Create order
        DECLARE @OrderId INT;
        
        INSERT INTO dbo.Order (CustomerId, OrderDate, Status)
        VALUES (@CustomerId, SYSDATETIME(), 'Pending');
        
        SET @OrderId = SCOPE_IDENTITY();
        
        -- Add line items and decrement stock
        INSERT INTO dbo.OrderLineItem (OrderId, ProductId, Quantity, UnitPrice)
        SELECT @OrderId, ProductId, Quantity, UnitPrice
        FROM @Items;
        
        UPDATE p
        SET StockQuantity = StockQuantity - i.Quantity
        FROM dbo.Product p
        INNER JOIN @Items i ON p.ProductId = i.ProductId;
        
        COMMIT TRANSACTION;
        
        SELECT @OrderId AS OrderId;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        THROW;
    END CATCH;
END;
```

## Deadlock Prevention

Access tables in consistent order:

```sql
-- ❌ Bad: Different procedures access tables in different order
CREATE PROCEDURE dbo.usp_UpdateOrderThenCustomer
AS
BEGIN
    BEGIN TRANSACTION;
    UPDATE dbo.Order WITH (UPDLOCK) WHERE OrderId = @OrderId;
    UPDATE dbo.Customer WITH (UPDLOCK) WHERE CustomerId = @CustomerId;
    COMMIT;
END;

CREATE PROCEDURE dbo.usp_UpdateCustomerThenOrder
AS
BEGIN
    BEGIN TRANSACTION;
    UPDATE dbo.Customer WITH (UPDLOCK) WHERE CustomerId = @CustomerId;
    UPDATE dbo.Order WITH (UPDLOCK) WHERE OrderId = @OrderId;
    COMMIT;
END;
-- ⚠️ These can deadlock each other

-- ✅ Good: Always access in same order (alphabetical)
CREATE PROCEDURE dbo.usp_UpdateBoth
AS
BEGIN
    BEGIN TRANSACTION;
    -- Always Customer first, then Order
    UPDATE dbo.Customer WITH (UPDLOCK) WHERE CustomerId = @CustomerId;
    UPDATE dbo.Order WITH (UPDLOCK) WHERE OrderId = @OrderId;
    COMMIT;
END;
```

## Monitoring Locks

```sql
-- View current locks
SELECT 
    tl.resource_type,
    tl.resource_database_id,
    tl.resource_associated_entity_id,
    tl.request_mode,
    tl.request_type,
    tl.request_status,
    tl.request_session_id,
    wt.blocking_session_id
FROM sys.dm_tran_locks AS tl
LEFT JOIN sys.dm_os_waiting_tasks AS wt 
    ON tl.lock_owner_address = wt.resource_address
WHERE tl.resource_database_id = DB_ID();

-- Find blocking sessions
SELECT 
    blocking.session_id AS BlockingSessionId,
    blocked.session_id AS BlockedSessionId,
    blocking_text.text AS BlockingSQL,
    blocked_text.text AS BlockedSQL
FROM sys.dm_exec_requests blocked
INNER JOIN sys.dm_exec_requests blocking 
    ON blocked.blocking_session_id = blocking.session_id
CROSS APPLY sys.dm_exec_sql_text(blocking.sql_handle) AS blocking_text
CROSS APPLY sys.dm_exec_sql_text(blocked.sql_handle) AS blocked_text
WHERE blocked.blocking_session_id <> 0;
```

## Why Race Conditions Are a Problem

1. **Data corruption**: Multiple updates lead to invalid state (negative inventory)
2. **Lost updates**: Last write wins, intermediate changes disappear
3. **Business rule violations**: Overbooking, double-booking, overselling
4. **Poor user experience**: Operations succeed then fail on conflict

## Symptoms

- Inventory going negative
- Duplicate bookings despite "available" checks
- Users reporting "it said available, then failed"
- Race condition bugs in concurrent scenarios

## Benefits

- **Guaranteed success**: First request locks, others wait
- **No retries needed**: Operation succeeds on first attempt
- **Predictable behavior**: Clear ordering of operations
- **Simpler error handling**: No conflict resolution logic

## Trade-offs

- **Reduced concurrency**: Locks block other transactions
- **Potential deadlocks**: Must access resources in consistent order
- **Performance impact**: Lock contention increases latency
- **Timeout handling**: Must handle lock timeout scenarios

## When to Use

✅ Use pessimistic locking when:
- High conflict probability (many users updating same data)
- Must succeed on first try (seat booking, inventory)
- Retry logic unacceptable to users
- Simple validation before update

❌ Don't use when:
- Low conflict rate (use optimistic concurrency)
- Long-running transactions (hold locks too long)
- Read-heavy workloads (lock-free reads preferred)

## See Also

- [Optimistic Concurrency](./optimistic-concurrency.md)
- [Explicit Transactions](./explicit-transactions.md)
- [Idempotent Operations](./idempotent-operations.md)
