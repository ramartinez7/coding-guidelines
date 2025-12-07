# User-Defined Types

> Create custom types for domain concepts used across multiple tables—ensure consistency and reduce duplication.

## Problem

The same domain concepts appear in many tables but are defined differently each time. Changes require updating multiple table definitions, and inconsistencies creep in over time.

## Example

### ❌ Before (Inline Type Definitions)

```sql
-- Email defined differently in each table
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,  -- 255 characters
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    Email NVARCHAR(100) NOT NULL,  -- ❌ Only 100 characters!
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId)
);

CREATE TABLE dbo.Vendor (
    VendorId INT NOT NULL,
    ContactEmail VARCHAR(200) NOT NULL,  -- ❌ VARCHAR instead of NVARCHAR!
    
    CONSTRAINT PK_Vendor PRIMARY KEY (VendorId)
);

-- Money defined inconsistently
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);

CREATE TABLE dbo.Invoice (
    InvoiceId INT NOT NULL,
    Total MONEY NOT NULL,  -- ❌ Different type!
    
    CONSTRAINT PK_Invoice PRIMARY KEY (InvoiceId)
);
```

### ✅ After (User-Defined Types)

```sql
-- Define domain types once
CREATE TYPE dbo.EmailAddress FROM NVARCHAR(255) NOT NULL;
GO

CREATE TYPE dbo.Money FROM DECIMAL(18,2) NOT NULL;
GO

CREATE TYPE dbo.Percentage FROM DECIMAL(5,2) NOT NULL;
GO

-- Use consistently across tables
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email dbo.EmailAddress,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    Email dbo.EmailAddress,  -- ✅ Same definition
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId)
);

CREATE TABLE dbo.Vendor (
    VendorId INT NOT NULL,
    ContactEmail dbo.EmailAddress,  -- ✅ Same definition
    
    CONSTRAINT PK_Vendor PRIMARY KEY (VendorId)
);

CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Price dbo.Money,  -- ✅ Consistent
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);

CREATE TABLE dbo.Invoice (
    InvoiceId INT NOT NULL,
    Total dbo.Money,  -- ✅ Consistent
    
    CONSTRAINT PK_Invoice PRIMARY KEY (InvoiceId)
);
```

## Alias Types (CREATE TYPE ... FROM)

```sql
-- Common domain types
CREATE TYPE dbo.EmailAddress FROM NVARCHAR(255) NOT NULL;
GO

CREATE TYPE dbo.PhoneNumber FROM VARCHAR(20) NULL;
GO

CREATE TYPE dbo.PostalCode FROM VARCHAR(10) NULL;
GO

CREATE TYPE dbo.Money FROM DECIMAL(18,2) NOT NULL;
GO

CREATE TYPE dbo.Percentage FROM DECIMAL(5,2) NOT NULL;
GO

CREATE TYPE dbo.Url FROM NVARCHAR(2000) NOT NULL;
GO

CREATE TYPE dbo.Description FROM NVARCHAR(MAX) NOT NULL;
GO

-- Use in tables
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email dbo.EmailAddress,
    Phone dbo.PhoneNumber,
    PostalCode dbo.PostalCode,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);
```

## Nullability with User-Defined Types

```sql
-- Type itself can define nullability
CREATE TYPE dbo.EmailAddress FROM NVARCHAR(255) NOT NULL;
GO

CREATE TYPE dbo.OptionalEmail FROM NVARCHAR(255) NULL;
GO

-- Or override at column level
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    -- Type is NOT NULL, column inherits it
    PrimaryEmail dbo.EmailAddress,
    -- Override to allow NULL
    SecondaryEmail dbo.EmailAddress NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);
```

## Types with Default Constraints

User-defined types cannot have defaults, but columns can:

```sql
CREATE TYPE dbo.Money FROM DECIMAL(18,2) NOT NULL;
GO

CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Price dbo.Money,
    -- Add default at column level
    ShippingCost dbo.Money DEFAULT 0,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);
```

## Business Domain Types

```sql
-- Currency code (ISO 4217)
CREATE TYPE dbo.CurrencyCode FROM CHAR(3) NOT NULL;
GO

-- Country code (ISO 3166-1 alpha-2)
CREATE TYPE dbo.CountryCode FROM CHAR(2) NOT NULL;
GO

-- Strongly-typed IDs
CREATE TYPE dbo.CustomerId FROM INT NOT NULL;
GO

CREATE TYPE dbo.OrderId FROM INT NOT NULL;
GO

CREATE TYPE dbo.ProductId FROM INT NOT NULL;
GO

-- Use in tables
CREATE TABLE dbo.Order (
    OrderId dbo.OrderId,
    CustomerId dbo.CustomerId,
    Total dbo.Money,
    CurrencyCode dbo.CurrencyCode,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId)
);
```

## Changing User-Defined Types

```sql
-- Cannot alter UDT if in use
-- Must drop and recreate

-- Find all dependencies
SELECT 
    OBJECT_SCHEMA_NAME(referencing_id) + '.' + 
    OBJECT_NAME(referencing_id) AS ReferencingObject,
    t.name AS UDTName
FROM sys.sql_expression_dependencies d
INNER JOIN sys.types t ON d.referenced_id = t.user_type_id
WHERE t.name = 'EmailAddress';

-- Drop columns using the type
ALTER TABLE dbo.Customer DROP COLUMN Email;
ALTER TABLE dbo.Employee DROP COLUMN Email;

-- Drop the type
DROP TYPE dbo.EmailAddress;
GO

-- Recreate with new definition
CREATE TYPE dbo.EmailAddress FROM NVARCHAR(320) NOT NULL;  -- Updated to RFC 5321 max
GO

-- Recreate columns
ALTER TABLE dbo.Customer ADD Email dbo.EmailAddress;
ALTER TABLE dbo.Employee ADD Email dbo.EmailAddress;
```

## Table-Valued Types

For passing structured data to stored procedures:

```sql
-- Define table-valued type
CREATE TYPE dbo.OrderItemType AS TABLE (
    ProductId INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(18,2) NOT NULL,
    PRIMARY KEY (ProductId)
);
GO

-- Use in stored procedure
CREATE PROCEDURE dbo.usp_CreateOrder
    @CustomerId INT,
    @Items dbo.OrderItemType READONLY  -- Must be READONLY
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Create order
        DECLARE @OrderId INT;
        
        INSERT INTO dbo.Order (CustomerId, OrderDate, Status)
        VALUES (@CustomerId, SYSDATETIME(), 'Pending');
        
        SET @OrderId = SCOPE_IDENTITY();
        
        -- Insert line items from TVP
        INSERT INTO dbo.OrderLineItem (OrderId, ProductId, Quantity, UnitPrice)
        SELECT @OrderId, ProductId, Quantity, UnitPrice
        FROM @Items;
        
        COMMIT TRANSACTION;
        
        SELECT @OrderId AS OrderId;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH;
END;
GO

-- Call procedure with structured data
DECLARE @Items dbo.OrderItemType;

INSERT INTO @Items (ProductId, Quantity, UnitPrice)
VALUES 
    (1, 2, 19.99),
    (2, 1, 49.99),
    (3, 5, 9.99);

EXEC dbo.usp_CreateOrder @CustomerId = 123, @Items = @Items;
```

## Complex Table-Valued Types

```sql
-- Address structure
CREATE TYPE dbo.AddressType AS TABLE (
    AddressId INT NOT NULL IDENTITY(1,1),
    Street NVARCHAR(200) NOT NULL,
    City NVARCHAR(100) NOT NULL,
    StateProvince NVARCHAR(50) NOT NULL,
    PostalCode VARCHAR(10) NOT NULL,
    CountryCode CHAR(2) NOT NULL,
    PRIMARY KEY (AddressId)
);
GO

-- Payment information
CREATE TYPE dbo.PaymentInfoType AS TABLE (
    PaymentMethod VARCHAR(20) NOT NULL,
    Amount DECIMAL(18,2) NOT NULL,
    CurrencyCode CHAR(3) NOT NULL,
    TransactionId VARCHAR(50) NULL,
    PRIMARY KEY (PaymentMethod)
);
GO

-- Batch update type
CREATE TYPE dbo.ProductUpdateType AS TABLE (
    ProductId INT NOT NULL,
    Price DECIMAL(18,2) NULL,
    StockQuantity INT NULL,
    Version INT NOT NULL,  -- For optimistic concurrency
    PRIMARY KEY (ProductId)
);
GO
```

## Using TVPs for Bulk Operations

```sql
-- Bulk insert helper type
CREATE TYPE dbo.ProductBulkType AS TABLE (
    SKU VARCHAR(50) NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    CategoryId INT NOT NULL,
    INDEX IX_SKU NONCLUSTERED (SKU)
);
GO

CREATE PROCEDURE dbo.usp_BulkInsertProducts
    @Products dbo.ProductBulkType READONLY
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Merge to handle duplicates
    MERGE dbo.Product AS target
    USING @Products AS source
    ON target.SKU = source.SKU
    WHEN MATCHED THEN
        UPDATE SET
            Name = source.Name,
            Price = source.Price,
            CategoryId = source.CategoryId
    WHEN NOT MATCHED THEN
        INSERT (SKU, Name, Price, CategoryId)
        VALUES (source.SKU, source.Name, source.Price, source.CategoryId);
END;
GO
```

## Viewing User-Defined Types

```sql
-- View all UDTs
SELECT 
    SCHEMA_NAME(schema_id) AS SchemaName,
    name AS TypeName,
    system_type_id,
    max_length,
    precision,
    scale,
    is_nullable
FROM sys.types
WHERE is_user_defined = 1
  AND is_table_type = 0
ORDER BY name;

-- View table-valued types
SELECT 
    SCHEMA_NAME(tt.schema_id) AS SchemaName,
    tt.name AS TypeName,
    c.name AS ColumnName,
    TYPE_NAME(c.system_type_id) AS DataType,
    c.max_length,
    c.precision,
    c.scale,
    c.is_nullable
FROM sys.table_types tt
INNER JOIN sys.columns c ON tt.type_table_object_id = c.object_id
ORDER BY tt.name, c.column_id;

-- Find where a type is used
SELECT 
    OBJECT_SCHEMA_NAME(c.object_id) + '.' + OBJECT_NAME(c.object_id) AS TableName,
    c.name AS ColumnName,
    t.name AS TypeName
FROM sys.columns c
INNER JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE t.is_user_defined = 1
ORDER BY t.name, TableName, ColumnName;
```

## Performance Considerations

```sql
-- TVPs are fast for small to medium batches
-- Better than individual inserts
DECLARE @Items dbo.OrderItemType;

-- ❌ Slow: Individual inserts (many round trips)
EXEC dbo.usp_AddOrderItem @OrderId = 1, @ProductId = 1, @Quantity = 2;
EXEC dbo.usp_AddOrderItem @OrderId = 1, @ProductId = 2, @Quantity = 1;
EXEC dbo.usp_AddOrderItem @OrderId = 1, @ProductId = 3, @Quantity = 5;

-- ✅ Fast: Single call with TVP
INSERT INTO @Items VALUES (1, 2, 19.99), (2, 1, 49.99), (3, 5, 9.99);
EXEC dbo.usp_CreateOrder @CustomerId = 123, @Items = @Items;

-- For very large batches (>10k rows), consider:
-- - BULK INSERT
-- - BCP utility  
-- - SqlBulkCopy in .NET
```

## Application Integration

### C# with ADO.NET

```csharp
using System.Data;
using System.Data.SqlClient;

// Create DataTable matching TVP structure
var items = new DataTable();
items.Columns.Add("ProductId", typeof(int));
items.Columns.Add("Quantity", typeof(int));
items.Columns.Add("UnitPrice", typeof(decimal));

// Add rows
items.Rows.Add(1, 2, 19.99);
items.Rows.Add(2, 1, 49.99);
items.Rows.Add(3, 5, 9.99);

// Pass to stored procedure
using var conn = new SqlConnection(connectionString);
using var cmd = new SqlCommand("dbo.usp_CreateOrder", conn);
cmd.CommandType = CommandType.StoredProcedure;
cmd.Parameters.AddWithValue("@CustomerId", 123);

var param = cmd.Parameters.AddWithValue("@Items", items);
param.SqlDbType = SqlDbType.Structured;
param.TypeName = "dbo.OrderItemType";

conn.Open();
var orderId = (int)cmd.ExecuteScalar();
```

### C# with Dapper

```csharp
using Dapper;

public class OrderItem
{
    public int ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

var items = new List<OrderItem>
{
    new OrderItem { ProductId = 1, Quantity = 2, UnitPrice = 19.99m },
    new OrderItem { ProductId = 2, Quantity = 1, UnitPrice = 49.99m },
    new OrderItem { ProductId = 3, Quantity = 5, UnitPrice = 9.99m }
};

var itemsTable = items.AsTableValuedParameter("dbo.OrderItemType",
    new[] { "ProductId", "Quantity", "UnitPrice" });

var orderId = await connection.QuerySingleAsync<int>(
    "dbo.usp_CreateOrder",
    new { CustomerId = 123, Items = itemsTable },
    commandType: CommandType.StoredProcedure
);
```

## Common Domain Types Library

```sql
-- String types
CREATE TYPE dbo.EmailAddress FROM NVARCHAR(255) NOT NULL;
CREATE TYPE dbo.PhoneNumber FROM VARCHAR(20) NULL;
CREATE TYPE dbo.Url FROM NVARCHAR(2000) NOT NULL;
CREATE TYPE dbo.Name FROM NVARCHAR(100) NOT NULL;
CREATE TYPE dbo.Description FROM NVARCHAR(MAX) NOT NULL;
GO

-- Numeric types
CREATE TYPE dbo.Money FROM DECIMAL(18,2) NOT NULL;
CREATE TYPE dbo.Percentage FROM DECIMAL(5,2) NOT NULL;
CREATE TYPE dbo.Quantity FROM INT NOT NULL;
GO

-- Code types
CREATE TYPE dbo.CountryCode FROM CHAR(2) NOT NULL;
CREATE TYPE dbo.CurrencyCode FROM CHAR(3) NOT NULL;
CREATE TYPE dbo.LanguageCode FROM CHAR(2) NOT NULL;
CREATE TYPE dbo.TimezoneCode FROM VARCHAR(50) NOT NULL;
GO

-- Identifier types
CREATE TYPE dbo.CustomerId FROM INT NOT NULL;
CREATE TYPE dbo.OrderId FROM INT NOT NULL;
CREATE TYPE dbo.ProductId FROM INT NOT NULL;
CREATE TYPE dbo.UserId FROM INT NOT NULL;
GO

-- Date/time types
CREATE TYPE dbo.Date FROM DATE NOT NULL;
CREATE TYPE dbo.Timestamp FROM DATETIME2(0) NOT NULL;
GO
```

## Why Inline Types Are a Problem

1. **Inconsistency**: Same concept defined differently in each table
2. **Maintenance**: Changing a type requires updating many tables
3. **Errors**: Easy to use wrong type (VARCHAR vs NVARCHAR, wrong length)
4. **No semantic meaning**: `NVARCHAR(255)` doesn't convey "email"
5. **Duplication**: Same constraints repeated everywhere

## Symptoms

- Different string lengths for same concept
- Mix of VARCHAR and NVARCHAR for same data
- Comments explaining what a column represents
- Inconsistent precision for money values

## Benefits

- **Consistency**: Same definition everywhere
- **Self-documenting**: Type name conveys meaning
- **Centralized changes**: Update type definition once
- **Type safety**: Wrong type won't compile (with TVPs)
- **Semantic compression**: `dbo.EmailAddress` vs `NVARCHAR(255)`

## Trade-offs

- **Less flexibility**: All uses must match exact type
- **Refactoring cost**: Changing UDT requires recreating columns
- **Learning curve**: Team must know custom types
- **Schema clutter**: More objects in database

## When to Use

✅ Use user-defined types for:
- Domain concepts used across multiple tables (Email, Money, etc.)
- Passing structured data to stored procedures (TVPs)
- Business-specific types (SKU, OrderNumber, etc.)
- Enforcing consistency across database

❌ Consider alternatives for:
- One-off columns unique to single table
- Types that change frequently
- External data with varying formats

## See Also

- [Domain-Appropriate Types](./domain-appropriate-types.md)
- [Schema Constraints](./schema-constraints.md)
- [Parameterized Queries](./parameterized-queries.md)
