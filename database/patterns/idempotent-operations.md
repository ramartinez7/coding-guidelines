# Idempotent Operations

> Design operations to be safely repeatable—use `MERGE` or conditional logic to prevent duplicate effects.

## Problem

Network failures, retries, and distributed systems mean database operations often execute multiple times. Non-idempotent operations create duplicate data or inconsistent state when retried.

## Example

### ❌ Before (Non-Idempotent)

```sql
-- Running twice creates duplicate orders
CREATE PROCEDURE dbo.usp_CreateOrder
    @OrderId INT,
    @CustomerId INT,
    @Total DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- ❌ Not idempotent: retry creates duplicate
    INSERT INTO dbo.Order (OrderId, CustomerId, OrderDate, Total, Status)
    VALUES (@OrderId, @CustomerId, SYSDATETIME(), @Total, 'Pending');
    
    -- If network fails after INSERT but before client receives response,
    -- client retries and creates duplicate order with same OrderId
END;

-- First call: Success, order 123 created
EXEC dbo.usp_CreateOrder @OrderId = 123, @CustomerId = 101, @Total = 99.99;

-- Network timeout, client retries
EXEC dbo.usp_CreateOrder @OrderId = 123, @CustomerId = 101, @Total = 99.99;
-- ❌ Error: Violation of PRIMARY KEY constraint
-- Or worse: creates order with different ID if OrderId is auto-generated
```

### ✅ After (Idempotent with MERGE)

```sql
-- Safe to run multiple times
CREATE PROCEDURE dbo.usp_CreateOrder
    @OrderId INT,
    @CustomerId INT,
    @Total DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- ✅ Idempotent: retry is safe
    MERGE dbo.Order AS target
    USING (SELECT @OrderId AS OrderId, @CustomerId AS CustomerId, @Total AS Total) AS source
    ON target.OrderId = source.OrderId
    WHEN NOT MATCHED THEN
        INSERT (OrderId, CustomerId, OrderDate, Total, Status)
        VALUES (source.OrderId, source.CustomerId, SYSDATETIME(), source.Total, 'Pending')
    WHEN MATCHED THEN
        -- Optionally update if values changed, or do nothing
        UPDATE SET Total = source.Total;  -- Or just leave WHEN MATCHED empty
    
    -- Returns OrderId whether created or already existed
    SELECT @OrderId AS OrderId;
END;

-- First call: Creates order
EXEC dbo.usp_CreateOrder @OrderId = 123, @CustomerId = 101, @Total = 99.99;

-- Retry: No error, no duplicate
EXEC dbo.usp_CreateOrder @OrderId = 123, @CustomerId = 101, @Total = 99.99;
-- ✅ Success: Returns OrderId 123, no duplicate created
```

### ✅ Alternative: IF NOT EXISTS

```sql
CREATE PROCEDURE dbo.usp_CreateOrder
    @OrderId INT,
    @CustomerId INT,
    @Total DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check if order already exists
    IF NOT EXISTS (SELECT 1 FROM dbo.Order WHERE OrderId = @OrderId)
    BEGIN
        INSERT INTO dbo.Order (OrderId, CustomerId, OrderDate, Total, Status)
        VALUES (@OrderId, @CustomerId, SYSDATETIME(), @Total, 'Pending');
    END;
    
    -- Return order whether created or already existed
    SELECT OrderId, CustomerId, OrderDate, Total, Status
    FROM dbo.Order
    WHERE OrderId = @OrderId;
END;
```

## Idempotent Updates

Use conditional updates to ensure idempotency:

```sql
-- ❌ Non-idempotent: increments multiple times
CREATE PROCEDURE dbo.usp_IncrementStock_Wrong
    @ProductId INT,
    @Quantity INT
AS
BEGIN
    UPDATE dbo.Product
    SET StockQuantity = StockQuantity + @Quantity
    WHERE ProductId = @ProductId;
END;

-- Called twice by accident, stock incremented twice
EXEC dbo.usp_IncrementStock_Wrong @ProductId = 1, @Quantity = 10;  -- Stock = 110
EXEC dbo.usp_IncrementStock_Wrong @ProductId = 1, @Quantity = 10;  -- Stock = 120 (wrong!)

-- ✅ Idempotent: use version number or request ID
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    StockQuantity INT NOT NULL,
    Version INT NOT NULL,  -- Optimistic concurrency
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);

CREATE PROCEDURE dbo.usp_IncrementStock
    @ProductId INT,
    @Quantity INT,
    @ExpectedVersion INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Only update if version matches
    UPDATE dbo.Product
    SET 
        StockQuantity = StockQuantity + @Quantity,
        Version = Version + 1
    WHERE ProductId = @ProductId
      AND Version = @ExpectedVersion;
    
    IF @@ROWCOUNT = 0
    BEGIN
        THROW 50001, 'Product version mismatch or not found', 1;
    END;
    
    -- Return updated product
    SELECT ProductId, StockQuantity, Version
    FROM dbo.Product
    WHERE ProductId = @ProductId;
END;

-- First call succeeds
EXEC dbo.usp_IncrementStock @ProductId = 1, @Quantity = 10, @ExpectedVersion = 1;
-- Version now 2

-- Retry with old version fails
EXEC dbo.usp_IncrementStock @ProductId = 1, @Quantity = 10, @ExpectedVersion = 1;
-- ❌ Error: version mismatch

-- Client must re-read and retry with new version
EXEC dbo.usp_IncrementStock @ProductId = 1, @Quantity = 10, @ExpectedVersion = 2;
```

## Using Idempotency Keys

For operations without natural keys, use client-provided idempotency keys:

```sql
CREATE TABLE dbo.Order (
    OrderId INT IDENTITY(1,1) NOT NULL,
    IdempotencyKey UNIQUEIDENTIFIER NOT NULL,  -- Client-provided
    CustomerId INT NOT NULL,
    OrderDate DATETIME2(0) NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT UQ_Order_IdempotencyKey UNIQUE (IdempotencyKey)
);

CREATE PROCEDURE dbo.usp_CreateOrder
    @IdempotencyKey UNIQUEIDENTIFIER,
    @CustomerId INT,
    @Total DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check if this idempotency key was already processed
    DECLARE @ExistingOrderId INT;
    
    SELECT @ExistingOrderId = OrderId
    FROM dbo.Order
    WHERE IdempotencyKey = @IdempotencyKey;
    
    IF @ExistingOrderId IS NOT NULL
    BEGIN
        -- Already processed, return existing order
        SELECT OrderId, CustomerId, OrderDate, Total, Status
        FROM dbo.Order
        WHERE OrderId = @ExistingOrderId;
        
        RETURN;
    END;
    
    -- Not processed yet, create new order
    INSERT INTO dbo.Order (IdempotencyKey, CustomerId, OrderDate, Total, Status)
    VALUES (@IdempotencyKey, @CustomerId, SYSDATETIME(), @Total, 'Pending');
    
    -- Return newly created order
    SELECT OrderId, CustomerId, OrderDate, Total, Status
    FROM dbo.Order
    WHERE OrderId = SCOPE_IDENTITY();
END;

-- Client generates idempotency key (e.g., UUID)
DECLARE @Key UNIQUEIDENTIFIER = NEWID();

-- First call creates order
EXEC dbo.usp_CreateOrder @IdempotencyKey = @Key, @CustomerId = 101, @Total = 99.99;

-- Retry returns same order, no duplicate
EXEC dbo.usp_CreateOrder @IdempotencyKey = @Key, @CustomerId = 101, @Total = 99.99;
```

## Idempotent Deletes

Deletes are naturally idempotent when using conditional logic:

```sql
-- ✅ Idempotent: no error if already deleted
CREATE PROCEDURE dbo.usp_DeleteOrder
    @OrderId INT,
    @UserId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Soft delete (naturally idempotent)
    UPDATE dbo.Order
    SET 
        DeletedAt = SYSDATETIME(),
        DeletedBy = @UserId
    WHERE OrderId = @OrderId
      AND DeletedAt IS NULL;  -- Only update if not already deleted
    
    -- Return affected rows (0 if already deleted, 1 if just deleted)
    SELECT @@ROWCOUNT AS RowsAffected;
END;

-- First call deletes
EXEC dbo.usp_DeleteOrder @OrderId = 123, @UserId = 1;  -- RowsAffected = 1

-- Retry does nothing
EXEC dbo.usp_DeleteOrder @OrderId = 123, @UserId = 1;  -- RowsAffected = 0
```

## State Transitions with Idempotency

Ensure state transitions are idempotent:

```sql
CREATE PROCEDURE dbo.usp_ShipOrder
    @OrderId INT,
    @TrackingNumber VARCHAR(50),
    @UserId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Update order status (idempotent: only if currently in Pending status)
        UPDATE dbo.Order
        SET 
            Status = 'Shipped',
            ShippedAt = SYSDATETIME(),
            TrackingNumber = @TrackingNumber,
            ModifiedBy = @UserId,
            ModifiedAt = SYSDATETIME()
        WHERE OrderId = @OrderId
          AND Status = 'Pending';  -- Only transition from Pending
        
        IF @@ROWCOUNT = 0
        BEGIN
            -- Check if already shipped with same tracking number (idempotent)
            IF EXISTS (
                SELECT 1 FROM dbo.Order 
                WHERE OrderId = @OrderId 
                  AND Status = 'Shipped'
                  AND TrackingNumber = @TrackingNumber
            )
            BEGIN
                -- Already shipped with same tracking, that's OK (idempotent)
                COMMIT TRANSACTION;
                
                SELECT OrderId, Status, ShippedAt, TrackingNumber
                FROM dbo.Order
                WHERE OrderId = @OrderId;
                
                RETURN;
            END
            ELSE
            BEGIN
                THROW 50002, 'Order cannot be shipped (invalid state or tracking mismatch)', 1;
            END;
        END;
        
        -- Log the event (idempotent: check if already logged)
        IF NOT EXISTS (
            SELECT 1 FROM dbo.OrderHistory 
            WHERE OrderId = @OrderId 
              AND Action = 'Shipped'
              AND TrackingNumber = @TrackingNumber
        )
        BEGIN
            INSERT INTO dbo.OrderHistory (OrderId, Action, TrackingNumber, CreatedBy, CreatedAt)
            VALUES (@OrderId, 'Shipped', @TrackingNumber, @UserId, SYSDATETIME());
        END;
        
        COMMIT TRANSACTION;
        
        SELECT OrderId, Status, ShippedAt, TrackingNumber
        FROM dbo.Order
        WHERE OrderId = @OrderId;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        THROW;
    END CATCH;
END;
```

## Why It's a Problem

1. **Duplicate data**: Retries create duplicate records
2. **Incorrect state**: Operations applied multiple times produce wrong results
3. **Data corruption**: Non-idempotent updates cause inconsistent state
4. **Hard to debug**: Issues only appear under retry conditions (race conditions)

## Symptoms

- Duplicate records with slightly different timestamps
- Primary key violations on retries
- Balance or count discrepancies (values incremented multiple times)
- Inconsistent state after network failures

## Benefits

- **Safe retries**: Operations can be safely repeated without side effects
- **Distributed systems**: Essential for reliable microservices and event-driven architectures
- **Simpler clients**: Applications don't need complex retry logic
- **Data consistency**: State remains consistent even under failure conditions

## See Also

- [Optimistic Concurrency](./optimistic-concurrency.md)
- [Explicit Transactions](./explicit-transactions.md)
- [Event Sourcing](./event-sourcing.md)
