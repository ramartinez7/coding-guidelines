# Filtered Indexes

> Create indexes on subsets of data—improve query performance while reducing index maintenance overhead and storage.

## Problem

Full table indexes consume storage and slow down writes even when queries only access a small subset of rows. Indexing all rows wastes resources when most queries filter by specific conditions.

## Example

### ❌ Before (Full Table Index)

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    OrderDate DATETIME2(0) NOT NULL,
    Status VARCHAR(20) NOT NULL,  -- 'Pending', 'Processing', 'Shipped', 'Delivered', 'Cancelled'
    Total DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId)
);

-- Index all rows, even though most orders are delivered/cancelled (historical)
CREATE NONCLUSTERED INDEX IX_Order_Status
ON dbo.Order(Status);

-- Problem: Index includes millions of historical rows
-- Most queries only care about active orders:
SELECT OrderId, Total
FROM dbo.Order
WHERE Status IN ('Pending', 'Processing', 'Shipped');
-- ❌ Index includes 10 million delivered/cancelled orders (wasted space)
```

### ✅ After (Filtered Index)

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    OrderDate DATETIME2(0) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId)
);

-- Index only active orders
CREATE NONCLUSTERED INDEX IX_Order_Status_Active
ON dbo.Order(Status)
WHERE Status IN ('Pending', 'Processing', 'Shipped');  -- Filter predicate

-- Benefits:
-- ✅ Smaller index (only 100K active orders vs 10M total)
-- ✅ Faster writes (don't update index for delivered/cancelled)
-- ✅ Better cache utilization (smaller index fits in memory)
```

## Common Scenarios

### Active Records Only

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    IsActive BIT NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

-- Index only active customers
CREATE NONCLUSTERED INDEX IX_Customer_Email_Active
ON dbo.Customer(Email)
WHERE IsActive = 1;

-- Most queries filter by active
SELECT CustomerId, Name
FROM dbo.Customer
WHERE Email = 'alice@example.com'
  AND IsActive = 1;  -- Uses filtered index
```

### Non-NULL Values

```sql
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    ManagerId INT NULL,  -- Most employees have managers, few don't
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId)
);

-- Index only employees with managers
CREATE NONCLUSTERED INDEX IX_Employee_ManagerId
ON dbo.Employee(ManagerId)
WHERE ManagerId IS NOT NULL;

-- Find direct reports
SELECT EmployeeId, FirstName, LastName
FROM dbo.Employee
WHERE ManagerId = 100;  -- Uses filtered index
```

### Recent Data

```sql
CREATE TABLE dbo.Transaction (
    TransactionId BIGINT NOT NULL,
    AccountId INT NOT NULL,
    TransactionDate DATETIME2(0) NOT NULL,
    Amount DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Transaction PRIMARY KEY (TransactionId)
);

-- Index only recent transactions (last 90 days)
CREATE NONCLUSTERED INDEX IX_Transaction_AccountId_Recent
ON dbo.Transaction(AccountId, TransactionDate)
WHERE TransactionDate >= DATEADD(DAY, -90, SYSDATETIME());

-- Most queries access recent data
SELECT TransactionId, Amount
FROM dbo.Transaction
WHERE AccountId = 123
  AND TransactionDate >= DATEADD(DAY, -30, SYSDATETIME());
-- Uses filtered index (subset of last 90 days)
```

### Sparse Columns

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    OptionalAttribute1 NVARCHAR(100) SPARSE NULL,  -- Only 5% of products have this
    OptionalAttribute2 NVARCHAR(100) SPARSE NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);

-- Index only rows with the optional attribute
CREATE NONCLUSTERED INDEX IX_Product_OptionalAttribute1
ON dbo.Product(OptionalAttribute1)
WHERE OptionalAttribute1 IS NOT NULL;

-- Query for products with this attribute
SELECT ProductId, Name
FROM dbo.Product
WHERE OptionalAttribute1 = 'SpecialValue';
-- Uses small filtered index instead of scanning millions of NULLs
```

### Soft Deletes

```sql
CREATE TABLE dbo.User (
    UserId INT NOT NULL,
    Username NVARCHAR(50) NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    DeletedAt DATETIME2(0) NULL,
    
    CONSTRAINT PK_User PRIMARY KEY (UserId)
);

-- Index only active users (not soft-deleted)
CREATE UNIQUE NONCLUSTERED INDEX UQ_User_Username_Active
ON dbo.User(Username)
WHERE DeletedAt IS NULL;

CREATE UNIQUE NONCLUSTERED INDEX UQ_User_Email_Active
ON dbo.User(Email)
WHERE DeletedAt IS NULL;

-- Enforce uniqueness only for active users
-- Allows reuse of username/email after soft delete
INSERT INTO dbo.User VALUES (1, 'alice', 'alice@example.com', NULL);  -- Active

UPDATE dbo.User SET DeletedAt = SYSDATETIME() WHERE UserId = 1;  -- Soft delete

INSERT INTO dbo.User VALUES (2, 'alice', 'alice@example.com', NULL);  -- OK (previous deleted)
```

### High-Value Customers

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    TotalPurchases DECIMAL(18,2) NOT NULL,
    IsVIP BIT NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

-- Index only VIP customers for priority queries
CREATE NONCLUSTERED INDEX IX_Customer_VIP
ON dbo.Customer(Name)
WHERE IsVIP = 1;

-- VIP customer lookup
SELECT CustomerId, TotalPurchases
FROM dbo.Customer
WHERE Name LIKE 'Smith%'
  AND IsVIP = 1;  -- Uses small filtered index
```

## Filtered Index with Includes

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    OrderDate DATETIME2(0) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    ShippingAddress NVARCHAR(500) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId)
);

-- Covering filtered index for active orders
CREATE NONCLUSTERED INDEX IX_Order_CustomerId_Active_Covering
ON dbo.Order(CustomerId, OrderDate)
INCLUDE (Total, ShippingAddress)
WHERE Status IN ('Pending', 'Processing', 'Shipped');

-- Covers entire query for active orders
SELECT OrderId, OrderDate, Total, ShippingAddress
FROM dbo.Order
WHERE CustomerId = 123
  AND Status IN ('Pending', 'Processing', 'Shipped')
ORDER BY OrderDate DESC;
-- Index seek + no key lookup (fully covered)
```

## Composite Filters

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    CategoryId INT NOT NULL,
    IsActive BIT NOT NULL,
    IsOnSale BIT NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);

-- Index only active products currently on sale
CREATE NONCLUSTERED INDEX IX_Product_Sale
ON dbo.Product(CategoryId, Price)
WHERE IsActive = 1 AND IsOnSale = 1;

-- Query for active sale items
SELECT ProductId, Price
FROM dbo.Product
WHERE CategoryId = 5
  AND IsActive = 1
  AND IsOnSale = 1
ORDER BY Price;
-- Uses small filtered index (fraction of all products)
```

## Filtered Indexes with Date Ranges

```sql
CREATE TABLE dbo.SensorReading (
    ReadingId BIGINT NOT NULL,
    SensorId INT NOT NULL,
    ReadingDate DATETIME2(0) NOT NULL,
    Value DECIMAL(10,4) NOT NULL,
    IsAnomaly BIT NOT NULL,
    
    CONSTRAINT PK_SensorReading PRIMARY KEY (ReadingId)
);

-- Separate indexes for different time periods
CREATE NONCLUSTERED INDEX IX_SensorReading_Current
ON dbo.SensorReading(SensorId, ReadingDate)
WHERE ReadingDate >= '2024-01-01';

CREATE NONCLUSTERED INDEX IX_SensorReading_Anomalies
ON dbo.SensorReading(SensorId, ReadingDate)
WHERE IsAnomaly = 1;

-- Query uses appropriate filtered index
SELECT AVG(Value)
FROM dbo.SensorReading
WHERE SensorId = 100
  AND ReadingDate >= '2024-01-01';
-- Uses IX_SensorReading_Current (smaller than full table)
```

## Filtered Indexes for Partitioned Tables

```sql
-- Alternative to partitioning for simple cases
CREATE TABLE dbo.LogEntry (
    LogId BIGINT NOT NULL,
    LogDate DATETIME2(0) NOT NULL,
    Severity VARCHAR(20) NOT NULL,
    Message NVARCHAR(MAX) NOT NULL,
    
    CONSTRAINT PK_LogEntry PRIMARY KEY (LogId)
);

-- Create filtered indexes by date range (pseudo-partitioning)
CREATE NONCLUSTERED INDEX IX_LogEntry_2024Q1
ON dbo.LogEntry(LogDate, Severity)
WHERE LogDate >= '2024-01-01' AND LogDate < '2024-04-01';

CREATE NONCLUSTERED INDEX IX_LogEntry_2024Q2
ON dbo.LogEntry(LogDate, Severity)
WHERE LogDate >= '2024-04-01' AND LogDate < '2024-07-01';

-- Optimizer chooses appropriate index based on date filter
SELECT * FROM dbo.LogEntry
WHERE LogDate >= '2024-02-01' AND LogDate < '2024-02-29'
  AND Severity = 'Error';
-- Uses IX_LogEntry_2024Q1
```

## Maintaining Filtered Indexes

```sql
-- Filtered index automatically maintained
CREATE NONCLUSTERED INDEX IX_Order_Status_Active
ON dbo.Order(Status)
WHERE Status IN ('Pending', 'Processing', 'Shipped');

-- Insert active order
INSERT INTO dbo.Order (OrderId, CustomerId, Status, Total)
VALUES (1, 100, 'Pending', 99.99);
-- ✅ Added to filtered index

-- Insert completed order
INSERT INTO dbo.Order (OrderId, CustomerId, Status, Total)
VALUES (2, 100, 'Delivered', 49.99);
-- ✅ NOT added to filtered index (doesn't match filter)

-- Update changes status to delivered
UPDATE dbo.Order
SET Status = 'Delivered'
WHERE OrderId = 1;
-- ✅ Removed from filtered index

-- Update changes status to shipped
UPDATE dbo.Order
SET Status = 'Shipped'
WHERE OrderId = 2;
-- ✅ Added to filtered index
```

## Viewing Filtered Indexes

```sql
-- View all filtered indexes
SELECT 
    OBJECT_SCHEMA_NAME(i.object_id) + '.' + OBJECT_NAME(i.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc,
    i.filter_definition,
    ps.row_count,
    ps.used_page_count
FROM sys.indexes i
INNER JOIN sys.dm_db_partition_stats ps 
    ON i.object_id = ps.object_id 
    AND i.index_id = ps.index_id
WHERE i.has_filter = 1
  AND i.is_disabled = 0
ORDER BY TableName, IndexName;

-- Compare filtered vs non-filtered index size
SELECT 
    i.name AS IndexName,
    i.type_desc,
    CASE WHEN i.has_filter = 1 THEN 'Filtered' ELSE 'Non-Filtered' END AS IndexType,
    i.filter_definition,
    ps.row_count AS RowCount,
    ps.used_page_count * 8 / 1024 AS SizeMB
FROM sys.indexes i
INNER JOIN sys.dm_db_partition_stats ps 
    ON i.object_id = ps.object_id 
    AND i.index_id = ps.index_id
WHERE OBJECT_NAME(i.object_id) = 'Order'
ORDER BY IndexName;
```

## Query Must Match Filter

```sql
CREATE NONCLUSTERED INDEX IX_Order_Status_Active
ON dbo.Order(Status)
WHERE Status IN ('Pending', 'Processing', 'Shipped');

-- ✅ Uses filtered index (matches filter exactly)
SELECT * FROM dbo.Order
WHERE Status IN ('Pending', 'Processing', 'Shipped');

-- ✅ Uses filtered index (subset of filter)
SELECT * FROM dbo.Order
WHERE Status = 'Pending';

-- ❌ Cannot use filtered index (includes 'Delivered' outside filter)
SELECT * FROM dbo.Order
WHERE Status IN ('Pending', 'Delivered');

-- ❌ Cannot use filtered index (no Status filter)
SELECT * FROM dbo.Order
WHERE CustomerId = 123;
```

## Statistics on Filtered Indexes

```sql
-- Filtered indexes have filtered statistics
CREATE NONCLUSTERED INDEX IX_Product_Active
ON dbo.Product(CategoryId)
WHERE IsActive = 1;

-- Statistics only include active products
-- Helps optimizer with cardinality estimation

-- View statistics
DBCC SHOW_STATISTICS('dbo.Product', 'IX_Product_Active');

-- Update statistics
UPDATE STATISTICS dbo.Product IX_Product_Active;
```

## Limitations

```sql
-- Filtered index limitations:

-- ❌ Cannot use subqueries
-- CREATE INDEX ... WHERE CustomerId IN (SELECT CustomerId FROM VIP);

-- ❌ Cannot use computed columns in filter
-- CREATE INDEX ... WHERE TotalAmount > Price * Quantity;

-- ❌ Cannot reference other tables
-- CREATE INDEX ... WHERE EXISTS (SELECT 1 FROM Customer WHERE ...);

-- ❌ Cannot use CAST or CONVERT
-- CREATE INDEX ... WHERE CAST(OrderDate AS DATE) = '2024-01-01';

-- ✅ Can use:
-- - Column comparisons: WHERE Status = 'Active'
-- - IS NULL / IS NOT NULL: WHERE DeletedAt IS NULL
-- - IN clause: WHERE Status IN ('A', 'B', 'C')
-- - AND / OR: WHERE IsActive = 1 AND DeletedAt IS NULL
-- - Comparison operators: WHERE OrderDate >= '2024-01-01'
```

## Performance Impact

```sql
-- Measure index usage
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    i.filter_definition,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates,
    ps.row_count,
    ps.used_page_count * 8 / 1024 AS SizeMB
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
INNER JOIN sys.dm_db_partition_stats ps ON s.object_id = ps.object_id AND s.index_id = ps.index_id
WHERE s.database_id = DB_ID()
  AND i.has_filter = 1
ORDER BY s.user_seeks + s.user_scans + s.user_lookups DESC;
```

## Why Full Table Indexes Are a Problem

1. **Wasted storage**: Index includes rows never queried
2. **Slower writes**: Update index entries for irrelevant rows
3. **Cache pollution**: Large indexes evict useful data from memory
4. **Maintenance overhead**: Rebuild/reorganize includes unused rows
5. **Statistics skew**: Histogram includes all data, not just queried subset

## Symptoms

- Large indexes with low seek/scan rates
- Queries always filter by same condition
- Most table rows in historical/inactive state
- Index maintenance takes too long
- Memory pressure from large indexes

## Benefits

- **Smaller indexes**: Only index relevant rows
- **Faster writes**: Don't maintain index for filtered-out rows
- **Better caching**: Smaller indexes fit in memory
- **Accurate statistics**: Statistics reflect queried subset
- **Faster seeks**: Fewer index pages to traverse

## Trade-offs

- **Query compatibility**: Query predicate must match filter
- **Multiple indexes**: May need several filtered indexes for different filters
- **Complexity**: More indexes to understand and maintain
- **Not always beneficial**: Small tables or uniform query patterns

## When to Use

✅ Use filtered indexes when:
- Queries consistently filter by same condition
- Large percentage of rows never queried (historical, inactive)
- Soft deletes leave many "deleted" rows
- Only non-NULL values are queried
- Want to enforce uniqueness on subset (active records only)

❌ Don't use when:
- Table is small (< 100K rows)
- All rows queried equally
- Query predicates vary widely
- Filter condition changes frequently

## See Also

- [Covering Indexes](./covering-indexes.md)
- [Indexed Views](./indexed-views.md)
- [Soft Deletes](./soft-deletes.md)
- [Execution Plans](./execution-plans.md)
