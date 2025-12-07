# Explicit Nullability

> Every column should explicitly declare whether `NULL` is allowed—make optionality a conscious design decision.

## Problem

Columns that don't specify `NULL` or `NOT NULL` get default behavior (nullable in most cases), leading to ambiguity about whether `NULL` represents:
- Missing data that should be provided
- Optional data by design
- An error condition
- A special business state

## Example

### ❌ Before (Implicit Nullability)

```sql
-- Unclear which columns are required vs optional
CREATE TABLE dbo.Customer (
    CustomerId INT PRIMARY KEY,
    FirstName NVARCHAR(100),      -- Is this required?
    LastName NVARCHAR(100),       -- Is this required?
    Email NVARCHAR(255),          -- Is this required?
    Phone NVARCHAR(20),           -- Is this optional?
    MiddleName NVARCHAR(100),     -- Is this optional?
    CompanyName NVARCHAR(200)     -- Is this optional?
);

-- Results in ambiguous queries
SELECT * FROM dbo.Customer WHERE Phone IS NULL;
-- Does NULL mean "no phone" or "phone not yet entered"?
```

### ✅ After (Explicit Nullability)

```sql
-- Crystal clear: required fields are NOT NULL, optional are NULL
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,   -- Required
    LastName NVARCHAR(100) NOT NULL,    -- Required
    Email NVARCHAR(255) NOT NULL,       -- Required
    Phone NVARCHAR(20) NULL,            -- Optional: not everyone has a phone
    MiddleName NVARCHAR(100) NULL,      -- Optional: not everyone has middle name
    CompanyName NVARCHAR(200) NULL,     -- Optional: B2C customers don't have this
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId),
    CONSTRAINT UQ_Customer_Email UNIQUE (Email)
);

-- Now queries are clear
SELECT * FROM dbo.Customer WHERE Phone IS NULL;
-- "Customers who haven't provided a phone number"
```

## Design Guidelines

### Default to NOT NULL

Unless you have a specific business reason for allowing `NULL`, make columns required.

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,                      -- Always required
    CustomerId INT NOT NULL,                   -- Always required
    OrderDate DATETIME2(0) NOT NULL,           -- Always required
    Total DECIMAL(18,2) NOT NULL,              -- Always required
    Status VARCHAR(20) NOT NULL,               -- Always required
    
    -- Optional fields have clear business meaning for NULL
    ShippedAt DATETIME2(0) NULL,               -- NULL = not yet shipped
    TrackingNumber VARCHAR(50) NULL,           -- NULL = no tracking yet
    CancellationReason NVARCHAR(500) NULL,     -- NULL = not cancelled
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId)
);
```

### Use Defaults for "No Value Yet"

For columns that will eventually have a value, use defaults instead of `NULL`:

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    Status VARCHAR(20) NOT NULL,
    CreatedAt DATETIME2(0) NOT NULL,
    ModifiedAt DATETIME2(0) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT DF_Order_Status DEFAULT 'Pending' FOR Status,
    CONSTRAINT DF_Order_CreatedAt DEFAULT SYSDATETIME() FOR CreatedAt,
    CONSTRAINT DF_Order_ModifiedAt DEFAULT SYSDATETIME() FOR ModifiedAt
);

-- Defaults eliminate the need for NULL
INSERT INTO dbo.Order (OrderId, CustomerId)
VALUES (1, 101);
-- Status = 'Pending', CreatedAt and ModifiedAt = current time
```

## NULL as Business State

Sometimes `NULL` has explicit business meaning—document this clearly:

```sql
CREATE TABLE dbo.Subscription (
    SubscriptionId INT NOT NULL,
    CustomerId INT NOT NULL,
    StartDate DATE NOT NULL,
    EndDate DATE NULL,              -- NULL = active (no end date)
    CancelledAt DATETIME2(0) NULL,  -- NULL = not cancelled
    
    CONSTRAINT PK_Subscription PRIMARY KEY (SubscriptionId),
    CONSTRAINT CK_Subscription_EndDate CHECK (EndDate IS NULL OR EndDate >= StartDate)
);

-- Query active subscriptions
SELECT * FROM dbo.Subscription 
WHERE EndDate IS NULL OR EndDate >= CAST(GETDATE() AS DATE);

-- Query cancelled subscriptions
SELECT * FROM dbo.Subscription 
WHERE CancelledAt IS NOT NULL;
```

## NULL vs Empty String

Never use empty strings as a substitute for `NULL`:

```sql
-- ❌ Bad: empty string instead of NULL
CREATE TABLE dbo.Customer (
    Phone NVARCHAR(20) NOT NULL,  -- Uses '' for "no phone"
    MiddleName NVARCHAR(100) NOT NULL  -- Uses '' for "no middle name"
);

INSERT INTO dbo.Customer (Phone, MiddleName) VALUES ('', '');

-- Queries become messy
SELECT * FROM dbo.Customer WHERE Phone = '' OR Phone IS NULL;

-- ✅ Good: NULL means "not provided"
CREATE TABLE dbo.Customer (
    Phone NVARCHAR(20) NULL,       -- NULL = no phone provided
    MiddleName NVARCHAR(100) NULL  -- NULL = no middle name
);

INSERT INTO dbo.Customer (Phone, MiddleName) VALUES (NULL, NULL);

-- Clean queries
SELECT * FROM dbo.Customer WHERE Phone IS NULL;
```

## NULL in Computed Columns

Be careful with NULL in calculations:

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    BasePrice DECIMAL(18,2) NOT NULL,
    DiscountPercent DECIMAL(5,2) NULL,  -- NULL = no discount
    
    -- ❌ NULL propagates through calculation
    FinalPrice AS (BasePrice - (BasePrice * DiscountPercent / 100)),
    -- If DiscountPercent is NULL, FinalPrice becomes NULL!
    
    -- ✅ Handle NULL explicitly
    FinalPriceCorrect AS (BasePrice - (BasePrice * ISNULL(DiscountPercent, 0) / 100)),
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);
```

## Three-Valued Logic

Remember that SQL uses three-valued logic (TRUE, FALSE, UNKNOWN):

```sql
-- Comparison with NULL returns UNKNOWN (not FALSE!)
SELECT 'TRUE' WHERE 1 = 1;        -- Returns 'TRUE'
SELECT 'FALSE' WHERE 1 = 0;       -- Returns nothing
SELECT 'UNKNOWN' WHERE 1 = NULL;  -- Returns nothing (NULL = NULL is UNKNOWN)

-- This affects WHERE clauses
SELECT * FROM dbo.Customer WHERE CompanyName = 'Acme';     -- Returns customers with CompanyName = 'Acme'
SELECT * FROM dbo.Customer WHERE CompanyName <> 'Acme';    -- Does NOT include customers with CompanyName = NULL
SELECT * FROM dbo.Customer WHERE CompanyName IS NULL;      -- Returns customers with NULL CompanyName

-- To get all non-Acme customers including NULLs:
SELECT * FROM dbo.Customer 
WHERE CompanyName <> 'Acme' OR CompanyName IS NULL;
```

## ANSI NULL Behavior

Ensure consistent NULL handling across connections:

```sql
-- Set at database level (recommended)
ALTER DATABASE MyDatabase
SET ANSI_NULLS ON;

-- Or at session level
SET ANSI_NULLS ON;

-- With ANSI_NULLS ON:
WHERE column = NULL    -- Always returns no rows
WHERE column IS NULL   -- Correct way to check for NULL

-- With ANSI_NULLS OFF (deprecated, avoid):
WHERE column = NULL    -- Returns rows where column is NULL
```

## Why It's a Problem

1. **Ambiguity**: Developers guess whether `NULL` is allowed or an error
2. **Runtime failures**: Application crashes when encountering unexpected `NULL`
3. **Incorrect logic**: `WHERE column <> value` doesn't include `NULL` rows
4. **Data quality**: Hard to distinguish missing data from intentionally absent data

## Symptoms

- Columns without `NULL` or `NOT NULL` specification
- Application code checking for both `NULL` and empty string
- NullReferenceException or similar errors in applications
- Inconsistent behavior between columns (some allow NULL, others don't)

## Benefits

- **Self-documenting**: Schema clearly shows required vs optional fields
- **Compile-time safety**: Database rejects invalid inserts immediately
- **Clearer queries**: Obvious when to use `IS NULL` checks
- **Better data quality**: Can't accidentally leave required fields empty

## Trade-offs

- **Schema changes**: Adding `NOT NULL` to existing columns requires data cleanup
- **Strict validation**: Cannot temporarily store incomplete records (usually a good thing)

## Migration Strategy

When adding `NOT NULL` to existing columns:

```sql
-- 1. Find and fix NULL values
UPDATE dbo.Customer
SET Email = 'unknown@example.com'  -- Or other default/placeholder
WHERE Email IS NULL;

-- 2. Add the constraint
ALTER TABLE dbo.Customer
ALTER COLUMN Email NVARCHAR(255) NOT NULL;

-- Or with a default for future inserts
ALTER TABLE dbo.Customer
ADD CONSTRAINT DF_Customer_Email DEFAULT 'unknown@example.com' FOR Email;

ALTER TABLE dbo.Customer
ALTER COLUMN Email NVARCHAR(255) NOT NULL;
```

## See Also

- [Schema Constraints](./schema-constraints.md)
- [Domain-Appropriate Types](./domain-appropriate-types.md)
- [Check Constraints](./check-constraints.md)
