# Optimistic Concurrency

> Use row versions or timestamps to detect conflicts—allow concurrent reads while preventing lost updates.

## Problem

Multiple users updating the same record simultaneously can cause lost updates—the last write wins, overwriting previous changes without warning.

## Example

### ❌ Before (Lost Update Problem)

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    StockQuantity INT NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);

-- User A reads product
SELECT ProductId, Name, StockQuantity, Price
FROM dbo.Product
WHERE ProductId = 123;
-- StockQuantity = 100

-- User B reads same product (same time)
SELECT ProductId, Name, StockQuantity, Price
FROM dbo.Product
WHERE ProductId = 123;
-- StockQuantity = 100

-- User A updates price
UPDATE dbo.Product
SET Price = 19.99
WHERE ProductId = 123;
-- StockQuantity still 100, Price now 19.99

-- User B updates stock (overwrites A's price change!)
UPDATE dbo.Product
SET StockQuantity = 90
WHERE ProductId = 123;
-- ❌ Price reverts to original value! User A's change lost.
```

### ✅ After (Version-Based Optimistic Concurrency)

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    StockQuantity INT NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    Version INT NOT NULL,  -- Concurrency token
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);

-- User A reads product
SELECT ProductId, Name, StockQuantity, Price, Version
FROM dbo.Product
WHERE ProductId = 123;
-- StockQuantity = 100, Version = 1

-- User B reads same product
SELECT ProductId, Name, StockQuantity, Price, Version
FROM dbo.Product
WHERE ProductId = 123;
-- StockQuantity = 100, Version = 1

-- User A updates price
UPDATE dbo.Product
SET 
    Price = 19.99,
    Version = Version + 1
WHERE ProductId = 123
  AND Version = 1;  -- Only succeed if version matches
-- ✅ Success: 1 row affected, Version now 2

IF @@ROWCOUNT = 0
    THROW 50001, 'Concurrency conflict: record was modified by another user', 1;

-- User B tries to update stock with old version
UPDATE dbo.Product
SET 
    StockQuantity = 90,
    Version = Version + 1
WHERE ProductId = 123
  AND Version = 1;  -- Fails! Version is now 2
-- ❌ 0 rows affected

IF @@ROWCOUNT = 0
BEGIN
    THROW 50001, 'Concurrency conflict: record was modified by another user', 1;
    -- Application can re-read, show conflict to user, and retry
END;
```

## Stored Procedure Pattern

```sql
CREATE PROCEDURE dbo.usp_UpdateProduct
    @ProductId INT,
    @Name NVARCHAR(200),
    @StockQuantity INT,
    @Price DECIMAL(18,2),
    @ExpectedVersion INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update with version check
    UPDATE dbo.Product
    SET 
        Name = @Name,
        StockQuantity = @StockQuantity,
        Price = @Price,
        Version = Version + 1,
        ModifiedAt = SYSDATETIME()
    WHERE ProductId = @ProductId
      AND Version = @ExpectedVersion;
    
    IF @@ROWCOUNT = 0
    BEGIN
        -- Check if record exists
        IF NOT EXISTS (SELECT 1 FROM dbo.Product WHERE ProductId = @ProductId)
        BEGIN
            THROW 50002, 'Product not found', 1;
        END
        ELSE
        BEGIN
            THROW 50001, 'Concurrency conflict: product was modified by another user', 1;
        END;
    END;
    
    -- Return updated product
    SELECT ProductId, Name, StockQuantity, Price, Version
    FROM dbo.Product
    WHERE ProductId = @ProductId;
END;
```

## Using ROWVERSION

SQL Server's `ROWVERSION` (formerly `TIMESTAMP`) is more efficient than `INT`:

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    StockQuantity INT NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    RowVersion ROWVERSION NOT NULL,  -- Automatically updated by SQL Server
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);

-- SQL Server automatically increments RowVersion on every update
UPDATE dbo.Product
SET Price = 19.99
WHERE ProductId = 123;
-- RowVersion automatically changed

-- Check concurrency with RowVersion
UPDATE dbo.Product
SET 
    StockQuantity = 90,
    Price = 24.99
WHERE ProductId = 123
  AND RowVersion = @ExpectedRowVersion;

IF @@ROWCOUNT = 0
    THROW 50001, 'Concurrency conflict', 1;
```

## Handling Conflicts in Application Code

### C# with Entity Framework

```csharp
public async Task UpdateProductAsync(Product product)
{
    try
    {
        _context.Entry(product).State = EntityState.Modified;
        await _context.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException ex)
    {
        // Conflict occurred
        var entry = ex.Entries.Single();
        var databaseValues = await entry.GetDatabaseValuesAsync();
        
        if (databaseValues == null)
        {
            // Record was deleted
            throw new InvalidOperationException("Product was deleted by another user");
        }
        
        // Show user the conflict
        var databaseProduct = (Product)databaseValues.ToObject();
        throw new ConcurrencyException(
            $"Product was modified by another user. Current values: {databaseProduct}"
        );
    }
}
```

### C# with Dapper

```csharp
public async Task<bool> UpdateProductAsync(int productId, ProductDto dto, int expectedVersion)
{
    var sql = @"
        UPDATE Product
        SET 
            Name = @Name,
            StockQuantity = @StockQuantity,
            Price = @Price,
            Version = Version + 1
        WHERE ProductId = @ProductId
          AND Version = @ExpectedVersion";
    
    var rowsAffected = await connection.ExecuteAsync(sql, new
    {
        ProductId = productId,
        dto.Name,
        dto.StockQuantity,
        dto.Price,
        ExpectedVersion = expectedVersion
    });
    
    if (rowsAffected == 0)
    {
        // Check if record exists
        var exists = await connection.ExecuteScalarAsync<bool>(
            "SELECT CAST(CASE WHEN EXISTS(SELECT 1 FROM Product WHERE ProductId = @ProductId) THEN 1 ELSE 0 END AS BIT)",
            new { ProductId = productId }
        );
        
        if (!exists)
            throw new NotFoundException($"Product {productId} not found");
        else
            throw new ConcurrencyException("Product was modified by another user");
    }
    
    return true;
}
```

## Optimistic vs Pessimistic Locking

| Aspect | Optimistic | Pessimistic |
|--------|-----------|-------------|
| **When to use** | High read, low conflict | High conflict scenarios |
| **Locking** | No locks, detect conflicts | Lock rows during read |
| **Concurrency** | High (readers don't block) | Low (readers may block) |
| **Performance** | Better for reads | Better for avoiding retries |
| **Complexity** | Must handle conflicts | Simpler (no conflicts) |

```sql
-- Optimistic: Read without locking
SELECT ProductId, Name, Price, Version
FROM dbo.Product
WHERE ProductId = 123;

-- Pessimistic: Lock during read
SELECT ProductId, Name, Price
FROM dbo.Product WITH (UPDLOCK, HOLDLOCK)
WHERE ProductId = 123;
-- Other users blocked until transaction completes
```

## Partial Updates with Optimistic Concurrency

Update only changed fields while checking version:

```sql
CREATE PROCEDURE dbo.usp_UpdateProductPrice
    @ProductId INT,
    @Price DECIMAL(18,2),
    @ExpectedVersion INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update only price, preserve other fields
    UPDATE dbo.Product
    SET 
        Price = @Price,
        Version = Version + 1,
        ModifiedAt = SYSDATETIME()
    WHERE ProductId = @ProductId
      AND Version = @ExpectedVersion;
    
    IF @@ROWCOUNT = 0
        THROW 50001, 'Concurrency conflict', 1;
    
    SELECT ProductId, Name, StockQuantity, Price, Version
    FROM dbo.Product
    WHERE ProductId = @ProductId;
END;

-- Concurrent updates to different fields succeed
-- User A: Update price (Version 1 -> 2)
EXEC dbo.usp_UpdateProductPrice @ProductId = 123, @Price = 19.99, @ExpectedVersion = 1;

-- User B: Cannot update stock with old version
-- Must re-read and get Version 2, then succeed
```

## Last-Write-Wins Alternative

Sometimes conflicts should be ignored (cache-like behavior):

```sql
-- No version check, last write always wins
UPDATE dbo.Cache
SET 
    Value = @Value,
    ExpiresAt = DATEADD(MINUTE, 30, SYSDATETIME())
WHERE CacheKey = @Key;

-- Used for:
-- - Cache tables
-- - Counters that don't need precision
-- - Temporary data where conflicts don't matter
```

## Why Lost Updates Are a Problem

1. **Silent data loss**: Changes disappear without warning
2. **Inconsistent state**: Database has mix of old and new values
3. **Business logic errors**: Decisions based on stale data
4. **Audit trail breaks**: Cannot track who made which change

## Symptoms

- User complaints about "lost changes"
- Data mysteriously reverting to old values
- Concurrent users seeing inconsistent state
- No way to detect simultaneous modifications

## Benefits

- **Prevents lost updates**: Conflicts detected and handled
- **High concurrency**: Readers don't block writers
- **Explicit handling**: Application knows when conflicts occur
- **Performance**: No locks during reads

## Trade-offs

- **Retry logic**: Application must handle conflicts
- **User experience**: Must show conflict resolution UI
- **Additional column**: Requires version column in table
- **Not for all scenarios**: High conflict rates need pessimistic locking

## When to Use

✅ Use optimistic concurrency when:
- Read-heavy workloads
- Low conflict probability
- UI shows current values before update
- Can show conflict resolution to user

❌ Don't use when:
- Very high conflict rates (use pessimistic locking)
- Cannot show conflicts to user
- Must guarantee no retries (banking transactions)

## See Also

- [Pessimistic Locking](./pessimistic-locking.md)
- [Idempotent Operations](./idempotent-operations.md)
- [Explicit Transactions](./explicit-transactions.md)
