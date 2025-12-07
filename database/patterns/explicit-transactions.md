# Explicit Transactions

> Wrap related operations in explicit transactions—ensure atomicity of business operations.

## Problem

Without explicit transaction boundaries, each statement auto-commits independently. Multi-step operations can fail halfway through, leaving data in an inconsistent state.

## Example

### ❌ Before (Implicit Transactions)

```sql
-- Each statement auto-commits independently
-- Transfer money between accounts

-- Debit from Account A
UPDATE dbo.Account
SET Balance = Balance - 100
WHERE AccountId = 123;

-- ❌ CRASH HERE! Money deducted but not credited

-- Credit to Account B (never executed)
UPDATE dbo.Account
SET Balance = Balance + 100
WHERE AccountId = 456;

-- Result: Account A lost $100, Account B didn't gain it!
-- Data is inconsistent, money disappeared
```

### ✅ After (Explicit Transaction)

```sql
-- Wrap in transaction: all-or-nothing
BEGIN TRY
    BEGIN TRANSACTION;
    
    -- Debit from Account A
    UPDATE dbo.Account
    SET Balance = Balance - 100
    WHERE AccountId = 123;
    
    -- ❌ CRASH HERE! Transaction rolls back
    
    -- Credit to Account B
    UPDATE dbo.Account
    SET Balance = Balance + 100
    WHERE AccountId = 456;
    
    COMMIT TRANSACTION;
    -- Both updates succeed or both fail
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
    
    THROW;
END CATCH;
```

## Transaction Best Practices

### Keep Transactions Short

```sql
-- ❌ Bad: Long-running transaction
BEGIN TRANSACTION;

SELECT * FROM dbo.Order;  -- 1 million rows, takes 30 seconds
UPDATE dbo.Order SET Status = 'Processed' WHERE OrderId = 123;
-- Other users blocked for 30+ seconds!

COMMIT TRANSACTION;

-- ✅ Good: Short transaction
-- Read outside transaction
DECLARE @OrderData TABLE (OrderId INT, CustomerId INT, Total DECIMAL(18,2));

INSERT INTO @OrderData
SELECT OrderId, CustomerId, Total
FROM dbo.Order;

-- Transaction only for write
BEGIN TRANSACTION;

UPDATE dbo.Order 
SET Status = 'Processed' 
WHERE OrderId = 123;

COMMIT TRANSACTION;
-- Other users blocked for milliseconds
```

### Always Use TRY...CATCH

```sql
CREATE PROCEDURE dbo.usp_TransferMoney
    @FromAccountId INT,
    @ToAccountId INT,
    @Amount DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Validate sufficient balance
        DECLARE @FromBalance DECIMAL(18,2);
        
        SELECT @FromBalance = Balance
        FROM dbo.Account WITH (UPDLOCK)
        WHERE AccountId = @FromAccountId;
        
        IF @FromBalance < @Amount
        BEGIN
            THROW 50001, 'Insufficient funds', 1;
        END;
        
        -- Debit
        UPDATE dbo.Account
        SET Balance = Balance - @Amount
        WHERE AccountId = @FromAccountId;
        
        -- Credit
        UPDATE dbo.Account
        SET Balance = Balance + @Amount
        WHERE AccountId = @ToAccountId;
        
        -- Log transaction
        INSERT INTO dbo.TransactionLog (FromAccountId, ToAccountId, Amount, TransactionDate)
        VALUES (@FromAccountId, @ToAccountId, @Amount, SYSDATETIME());
        
        COMMIT TRANSACTION;
        
        SELECT 'SUCCESS' AS Result, @Amount AS Amount;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        -- Re-throw with context
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
        DECLARE @ErrorState INT = ERROR_STATE();
        
        RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
    END CATCH;
END;
```

## Nested Transactions

SQL Server doesn't support true nested transactions, but you can track depth:

```sql
CREATE PROCEDURE dbo.usp_OuterOperation
AS
BEGIN
    BEGIN TRANSACTION;  -- @@TRANCOUNT = 1
    
    UPDATE dbo.Table1 SET Column1 = 'A';
    
    EXEC dbo.usp_InnerOperation;  -- Calls BEGIN TRAN again
    
    COMMIT TRANSACTION;  -- @@TRANCOUNT back to 0
END;

CREATE PROCEDURE dbo.usp_InnerOperation
AS
BEGIN
    -- Check if already in transaction
    DECLARE @TranStarted BIT = 0;
    
    IF @@TRANCOUNT = 0
    BEGIN
        BEGIN TRANSACTION;
        SET @TranStarted = 1;
    END;
    ELSE
    BEGIN
        SAVE TRANSACTION InnerSavePoint;  -- Create savepoint
    END;
    
    BEGIN TRY
        UPDATE dbo.Table2 SET Column2 = 'B';
        
        IF @TranStarted = 1
            COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @TranStarted = 1
        BEGIN
            IF @@TRANCOUNT > 0
                ROLLBACK TRANSACTION;
        END
        ELSE
        BEGIN
            IF @@TRANCOUNT > 0
                ROLLBACK TRANSACTION InnerSavePoint;
        END;
        
        THROW;
    END CATCH;
END;
```

## Savepoints

Create checkpoints within transactions:

```sql
BEGIN TRANSACTION;

UPDATE dbo.Order SET Status = 'Processing' WHERE OrderId = 123;

SAVE TRANSACTION BeforeLineItems;

BEGIN TRY
    -- Insert line items
    INSERT INTO dbo.OrderLineItem (OrderId, ProductId, Quantity, UnitPrice)
    SELECT @OrderId, ProductId, Quantity, UnitPrice
    FROM @LineItems;
END TRY
BEGIN CATCH
    -- Rollback only line items, keep order update
    ROLLBACK TRANSACTION BeforeLineItems;
    
    -- Continue with transaction
END CATCH;

COMMIT TRANSACTION;
```

## Isolation Levels

Control how transactions interact:

```sql
-- Default: READ COMMITTED
-- Readers don't see uncommitted changes
-- Writers block readers until commit

-- READ UNCOMMITTED (dirty reads)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM dbo.Order;  -- May see uncommitted changes

-- REPEATABLE READ
-- Readers hold locks until transaction ends
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;
SELECT * FROM dbo.Order WHERE OrderId = 123;
-- Other transactions cannot modify this row until COMMIT
COMMIT TRANSACTION;

-- SERIALIZABLE
-- Prevents phantom reads (new rows appearing)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;
SELECT * FROM dbo.Order WHERE CustomerId = 101;
-- Other transactions cannot insert new orders for this customer
COMMIT TRANSACTION;

-- SNAPSHOT
-- Readers see consistent snapshot, no blocking
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
SELECT * FROM dbo.Order;  -- Sees snapshot at start of transaction
```

## Deadlock Handling

Prevent and handle deadlocks:

```sql
-- Deadlock prevention: access tables in consistent order
CREATE PROCEDURE dbo.usp_UpdateOrderAndCustomer_Safe
    @OrderId INT,
    @CustomerId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Always update Customer before Order (consistent ordering)
        UPDATE dbo.Customer
        SET ModifiedAt = SYSDATETIME()
        WHERE CustomerId = @CustomerId;
        
        UPDATE dbo.Order
        SET ModifiedAt = SYSDATETIME()
        WHERE OrderId = @OrderId;
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        -- Check for deadlock
        IF ERROR_NUMBER() = 1205  -- Deadlock victim
        BEGIN
            -- Retry logic or notify caller
            RAISERROR('Deadlock occurred, please retry', 16, 1);
        END
        ELSE
        BEGIN
            THROW;
        END;
    END CATCH;
END;
```

## Distributed Transactions

For operations across multiple databases:

```sql
-- Simple cross-database transaction
BEGIN TRANSACTION;

UPDATE Database1.dbo.Table1 SET Column1 = 'A';
UPDATE Database2.dbo.Table2 SET Column2 = 'B';

COMMIT TRANSACTION;

-- Linked server transaction
BEGIN DISTRIBUTED TRANSACTION;

UPDATE LocalDatabase.dbo.Table1 SET Column1 = 'A';
UPDATE LinkedServer.RemoteDatabase.dbo.Table2 SET Column2 = 'B';

COMMIT TRANSACTION;
```

## Transaction Logging

Log transaction outcomes:

```sql
CREATE PROCEDURE dbo.usp_ProcessOrder
    @OrderId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @TransactionId UNIQUEIDENTIFIER = NEWID();
    DECLARE @StartTime DATETIME2(3) = SYSDATETIME();
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Business logic
        UPDATE dbo.Order SET Status = 'Processed' WHERE OrderId = @OrderId;
        
        -- Log success
        INSERT INTO dbo.TransactionLog (
            TransactionId, 
            TransactionType, 
            Status, 
            StartTime, 
            EndTime
        )
        VALUES (
            @TransactionId,
            'ProcessOrder',
            'SUCCESS',
            @StartTime,
            SYSDATETIME()
        );
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        -- Log failure
        INSERT INTO dbo.TransactionLog (
            TransactionId,
            TransactionType,
            Status,
            ErrorMessage,
            StartTime,
            EndTime
        )
        VALUES (
            @TransactionId,
            'ProcessOrder',
            'FAILED',
            ERROR_MESSAGE(),
            @StartTime,
            SYSDATETIME()
        );
        
        THROW;
    END CATCH;
END;
```

## Checking Transaction State

```sql
-- Check if in transaction
IF @@TRANCOUNT > 0
BEGIN
    PRINT 'Inside a transaction';
END;

-- Check transaction isolation level
DBCC USEROPTIONS;

-- Monitor open transactions
SELECT 
    session_id,
    transaction_id,
    CASE transaction_state
        WHEN 0 THEN 'Not initialized'
        WHEN 1 THEN 'Initialized'
        WHEN 2 THEN 'Active'
        WHEN 3 THEN 'Ended (read-only)'
        WHEN 4 THEN 'Commit initiated'
        WHEN 5 THEN 'Prepared'
        WHEN 6 THEN 'Committed'
        WHEN 7 THEN 'Rolling back'
        WHEN 8 THEN 'Rolled back'
    END AS transaction_state_desc
FROM sys.dm_tran_session_transactions st
INNER JOIN sys.dm_tran_active_transactions at 
    ON st.transaction_id = at.transaction_id;
```

## Why Implicit Transactions Are a Problem

1. **Inconsistent state**: Multi-step operations can fail halfway
2. **No rollback**: Cannot undo changes when errors occur
3. **Race conditions**: Concurrent operations see partial updates
4. **Data corruption**: Business invariants violated

## Symptoms

- Data inconsistencies (e.g., debit without credit)
- "Orphaned" records with no parent
- Duplicate records from retry logic
- Constraint violations from partial operations

## Benefits

- **Atomicity**: All operations succeed or all fail
- **Consistency**: Database always in valid state
- **Error handling**: Automatic rollback on failure
- **Concurrency**: Proper isolation from other transactions

## Trade-offs

- **Locking**: Transactions hold locks, blocking other users
- **Deadlocks**: Possible when multiple transactions wait for each other
- **Performance**: Transaction overhead (minimal in practice)
- **Complexity**: Must handle rollbacks and savepoints

## See Also

- [Idempotent Operations](./idempotent-operations.md)
- [Optimistic Concurrency](./optimistic-concurrency.md)
- [Pessimistic Locking](./pessimistic-locking.md)
