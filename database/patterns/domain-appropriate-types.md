# Domain-Appropriate Types

> Use data types that match domain concepts—`DECIMAL` for money, `DATETIME2` for timestamps, not `VARCHAR` for everything.

## Problem

Using incorrect data types loses precision, semantic meaning, and database-level validation. `VARCHAR` stores everything but validates nothing; `FLOAT` loses precision; `DATETIME` has hidden `Kind` issues.

## Example

### ❌ Before (Wrong Types)

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name VARCHAR(MAX) NOT NULL,              -- Overkill for a name
    Price FLOAT NOT NULL,                     -- ❌ Loses precision for money
    CreatedDate VARCHAR(50) NOT NULL,         -- ❌ String for dates
    IsActive VARCHAR(5) NOT NULL,             -- ❌ String for boolean
    Quantity VARCHAR(20) NOT NULL,            -- ❌ String for numbers
    Tags VARCHAR(MAX) NOT NULL,               -- ❌ Comma-separated list
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);

-- Problems:
INSERT INTO dbo.Product (ProductId, Name, Price, CreatedDate, IsActive, Quantity, Tags)
VALUES (1, 'Widget', 19.99, '2024-01-15', 'true', '100', 'tag1,tag2,tag3');

-- 1. Price FLOAT: 19.99 becomes 19.989999... (precision loss)
-- 2. CreatedDate VARCHAR: '2024-01-15', 'Jan 15 2024', '01/15/2024' all valid (inconsistent)
-- 3. IsActive VARCHAR: 'true', 'True', 'yes', '1', 'Y' all valid (no consistency)
-- 4. Quantity VARCHAR: Can insert 'abc' (no validation)
-- 5. Tags VARCHAR: Cannot query individual tags efficiently
```

### ✅ After (Correct Types)

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,              -- ✅ Fixed length, Unicode
    Price DECIMAL(18,2) NOT NULL,             -- ✅ Exact precision for money
    CreatedDate DATETIME2(0) NOT NULL,        -- ✅ Proper date type
    IsActive BIT NOT NULL,                    -- ✅ Boolean type
    Quantity INT NOT NULL,                    -- ✅ Integer type
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId),
    CONSTRAINT CK_Product_Price CHECK (Price >= 0),
    CONSTRAINT CK_Product_Quantity CHECK (Quantity >= 0)
);

-- Tags in separate table (many-to-many)
CREATE TABLE dbo.ProductTag (
    ProductId INT NOT NULL,
    Tag NVARCHAR(50) NOT NULL,
    
    CONSTRAINT PK_ProductTag PRIMARY KEY (ProductId, Tag),
    CONSTRAINT FK_ProductTag_Product FOREIGN KEY (ProductId)
        REFERENCES dbo.Product(ProductId)
);

-- Benefits:
INSERT INTO dbo.Product (ProductId, Name, Price, CreatedDate, IsActive, Quantity)
VALUES (1, 'Widget', 19.99, '2024-01-15', 1, 100);

-- 1. Price exact: 19.99 stored precisely
-- 2. CreatedDate validated and sortable
-- 3. IsActive: only 0 or 1
-- 4. Quantity: only integers
-- 5. Tags queryable individually
```

## Type Guidelines

### Money and Decimals

```sql
-- ❌ Bad: FLOAT loses precision
Price FLOAT

INSERT ... VALUES (19.99);  -- Stored as 19.989999...
SELECT Price WHERE Price = 19.99;  -- May not find it!

-- ✅ Good: DECIMAL for exact precision
Price DECIMAL(18,2)  -- 18 digits, 2 after decimal

-- For money: DECIMAL(18,2) or DECIMAL(19,4)
-- For percentages: DECIMAL(5,2) (allows 0.00 to 100.00)
```

### Dates and Times

```sql
-- ❌ Bad: VARCHAR for dates
OrderDate VARCHAR(50)  -- 'Jan 15', '2024-01-15', '01/15/2024' all "valid"

-- ❌ Bad: DATETIME (old type, less precise, 8 bytes)
OrderDate DATETIME

-- ✅ Good: DATETIME2 (more precise, can be smaller)
OrderDate DATETIME2(0)  -- Second precision, 6 bytes
CreatedAt DATETIME2(3)  -- Millisecond precision, 7 bytes
Timestamp DATETIME2(7)  -- 100-nanosecond precision, 8 bytes

-- For dates only: DATE (3 bytes)
BirthDate DATE

-- For times only: TIME(0) (3 bytes)
BusinessHoursStart TIME(0)

-- For date+time with offset: DATETIMEOFFSET
EventTime DATETIMEOFFSET(0)  -- Includes timezone offset
```

### Booleans

```sql
-- ❌ Bad: VARCHAR for boolean
IsActive VARCHAR(5)  -- 'true', 'false', 'yes', 'no', '1', '0' all valid

-- ❌ Bad: INT for boolean
IsActive INT  -- 1, 0, 2, -1, 42 all "valid"

-- ✅ Good: BIT
IsActive BIT  -- Only 0 or 1

-- Multiple flags: separate BIT columns, not bitmask
IsActive BIT NOT NULL DEFAULT 1
IsDeleted BIT NOT NULL DEFAULT 0
IsVerified BIT NOT NULL DEFAULT 0
```

### Integers

```sql
-- ❌ Bad: VARCHAR for numbers
Quantity VARCHAR(20)  -- Can insert 'abc', no math operations

-- ✅ Good: Choose appropriate integer size
TINYINT    -- 0 to 255 (1 byte) - age, status codes
SMALLINT   -- -32,768 to 32,767 (2 bytes) - year, quantity
INT        -- -2B to 2B (4 bytes) - most IDs, quantities
BIGINT     -- -9 quintillion to 9 quintillion (8 bytes) - large IDs

-- Examples:
Age TINYINT CHECK (Age >= 0 AND Age <= 120)
Year SMALLINT
CustomerId INT
OrderLineItemId BIGINT
```

### Strings

```sql
-- Fixed-length (padded with spaces)
CHAR(10)        -- Fixed 10 bytes, for codes
NCHAR(10)       -- Fixed 10 Unicode chars, for codes

-- Variable-length (size based on content)
VARCHAR(100)    -- Up to 100 bytes, non-Unicode
NVARCHAR(100)   -- Up to 100 Unicode chars (international)

-- Large text
VARCHAR(MAX)    -- Up to 2GB, use sparingly
NVARCHAR(MAX)   -- Up to 1GB Unicode, use sparingly

-- Guidelines:
-- - Use NVARCHAR for international text
-- - Use fixed sizes (avoid MAX) when possible
-- - Choose realistic max length

FirstName NVARCHAR(100) NOT NULL
Email NVARCHAR(255) NOT NULL
Description NVARCHAR(2000) NOT NULL
Notes NVARCHAR(MAX) NULL  -- Only for truly unbounded text
```

### GUIDs

```sql
-- ✅ UNIQUEIDENTIFIER for GUIDs
OrderReference UNIQUEIDENTIFIER DEFAULT NEWID()

-- ❌ Don't store as VARCHAR(36)
OrderReference VARCHAR(36)  -- '123e4567-e89b-12d3-a456-426614174000'
-- Wastes space, loses validation
```

### JSON

```sql
-- Store JSON as NVARCHAR(MAX)
Metadata NVARCHAR(MAX) NOT NULL
    CONSTRAINT CK_Product_Metadata_Json CHECK (ISJSON(Metadata) = 1)

-- Query JSON fields
SELECT 
    ProductId,
    JSON_VALUE(Metadata, '$.manufacturer') AS Manufacturer,
    JSON_VALUE(Metadata, '$.warranty') AS Warranty
FROM dbo.Product;

-- For structured data, prefer separate columns/tables
-- JSON for truly flexible, schema-less data
```

### Binary Data

```sql
-- Small binary: BINARY(n) or VARBINARY(n)
PasswordHash BINARY(64)  -- SHA-512 hash (fixed size)
FileChecksum VARBINARY(32)  -- Variable up to 32 bytes

-- Large binary: VARBINARY(MAX)
FileContent VARBINARY(MAX)  -- Store files in DB (consider FileStream)

-- Images: Consider storing path in VARCHAR, file in blob storage
ProfileImagePath NVARCHAR(500)  -- URL or file path
-- Not: ProfileImage VARBINARY(MAX) (slow, large DB)
```

## User-Defined Types

Create reusable domain types:

```sql
-- Define once
CREATE TYPE dbo.EmailAddress FROM NVARCHAR(255) NOT NULL;
CREATE TYPE dbo.Money FROM DECIMAL(18,2) NOT NULL;
CREATE TYPE dbo.Percentage FROM DECIMAL(5,2) NOT NULL;
CREATE TYPE dbo.PhoneNumber FROM NVARCHAR(20) NOT NULL;

-- Use consistently
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email dbo.EmailAddress,
    Phone dbo.PhoneNumber NULL,
    CreditLimit dbo.Money,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Price dbo.Money,
    DiscountPercent dbo.Percentage,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);
```

## Common Mistakes

### Mistake 1: VARCHAR for Everything

```sql
-- ❌ Bad
CREATE TABLE dbo.Order (
    OrderId VARCHAR(50),
    CustomerId VARCHAR(50),
    OrderDate VARCHAR(50),
    Total VARCHAR(50),
    IsShipped VARCHAR(10)
);

-- ✅ Good
CREATE TABLE dbo.Order (
    OrderId INT,
    CustomerId INT,
    OrderDate DATETIME2(0),
    Total DECIMAL(18,2),
    IsShipped BIT
);
```

### Mistake 2: FLOAT for Money

```sql
-- ❌ Bad: Precision loss
Price FLOAT
INSERT ... VALUES (19.99);  -- Stored as 19.989999...

-- ✅ Good: Exact precision
Price DECIMAL(18,2)
INSERT ... VALUES (19.99);  -- Stored as 19.99
```

### Mistake 3: Wrong Size

```sql
-- ❌ Bad: Waste space
FirstName NVARCHAR(MAX)  -- 2GB max for a name?

-- ❌ Bad: Too small
FirstName NVARCHAR(10)  -- Truncates "Christopher"

-- ✅ Good: Appropriate size
FirstName NVARCHAR(100)  -- Fits most names
```

## Why Wrong Types Are a Problem

1. **Lost precision**: FLOAT for money loses cents
2. **No validation**: VARCHAR accepts any garbage
3. **Performance**: Wrong types prevent index use
4. **Storage waste**: NVARCHAR(MAX) for short strings
5. **Semantic loss**: Types don't communicate intent

## Symptoms

- `VARCHAR` or `NVARCHAR(MAX)` for everything
- `FLOAT` for monetary values
- Comments saying "store as string"
- `CAST`/`CONVERT` in every query
- Invalid data passing validation

## Benefits

- **Type safety**: Database validates data types
- **Performance**: Correct types enable efficient indexing
- **Storage efficiency**: Right-sized columns save space
- **Self-documenting**: Types communicate intent
- **Query optimization**: Optimizer uses type info

## See Also

- [Schema Constraints](./schema-constraints.md)
- [Explicit Nullability](./explicit-nullability.md)
- [User-Defined Types](./user-defined-types.md)
- [Check Constraints](./check-constraints.md)
