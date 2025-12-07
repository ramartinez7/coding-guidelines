# Unique Constraints

> Enforce uniqueness of values beyond the primary key—prevent duplicate data and model natural keys.

## Problem

Many business entities have natural unique identifiers beyond their primary key (email addresses, usernames, order numbers). Without unique constraints, duplicates can be inserted, breaking business rules.

## Example

### ❌ Before (No Uniqueness Enforced)

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    Username NVARCHAR(50) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

-- Insert customer
INSERT INTO dbo.Customer (CustomerId, Email, Username)
VALUES (1, 'alice@example.com', 'alice');

-- Insert duplicate email
INSERT INTO dbo.Customer (CustomerId, Email, Username)
VALUES (2, 'alice@example.com', 'alice2');
-- ❌ Succeeds! Duplicate email address
```

### ✅ After (Unique Constraint)

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    Username NVARCHAR(50) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId),
    CONSTRAINT UQ_Customer_Email UNIQUE (Email),
    CONSTRAINT UQ_Customer_Username UNIQUE (Username)
);

-- Insert customer
INSERT INTO dbo.Customer (CustomerId, Email, Username)
VALUES (1, 'alice@example.com', 'alice');
-- ✅ Success

-- Try duplicate email
INSERT INTO dbo.Customer (CustomerId, Email, Username)
VALUES (2, 'alice@example.com', 'alice2');
-- ❌ Error: Violation of UNIQUE KEY constraint 'UQ_Customer_Email'
```

## Single Column Uniqueness

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    SKU VARCHAR(50) NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId),
    CONSTRAINT UQ_Product_SKU UNIQUE (SKU)
);

-- Each SKU must be unique
INSERT INTO dbo.Product (ProductId, SKU, Name)
VALUES (1, 'WID-001', 'Widget');  -- OK

INSERT INTO dbo.Product (ProductId, SKU, Name)
VALUES (2, 'WID-001', 'Super Widget');  -- ❌ Error: Duplicate SKU
```

## Composite Unique Constraints

```sql
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    DepartmentId INT NOT NULL,
    EmployeeNumber VARCHAR(20) NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId),
    -- Employee number unique within department
    CONSTRAINT UQ_Employee_DeptNumber UNIQUE (DepartmentId, EmployeeNumber),
    -- Email globally unique
    CONSTRAINT UQ_Employee_Email UNIQUE (Email)
);

-- Same employee number OK in different departments
INSERT INTO dbo.Employee (EmployeeId, DepartmentId, EmployeeNumber, Email)
VALUES (1, 10, 'EMP-001', 'alice@example.com');  -- OK

INSERT INTO dbo.Employee (EmployeeId, DepartmentId, EmployeeNumber, Email)
VALUES (2, 20, 'EMP-001', 'bob@example.com');    -- OK (different dept)

INSERT INTO dbo.Employee (EmployeeId, DepartmentId, EmployeeNumber, Email)
VALUES (3, 10, 'EMP-001', 'carol@example.com');  -- ❌ Error: Duplicate in dept 10
```

## Unique Constraint vs Unique Index

```sql
-- Unique constraint (creates unique index automatically)
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId),
    CONSTRAINT UQ_Customer_Email UNIQUE (Email)
);

-- Equivalent to:
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

CREATE UNIQUE NONCLUSTERED INDEX UQ_Customer_Email
ON dbo.Customer(Email);

-- Difference: Constraint shows in schema as a constraint
-- Index shows as an index (both enforce uniqueness)
```

## Unique Constraints with NULL

```sql
-- UNIQUE allows multiple NULLs (NULL != NULL)
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    MiddleName NVARCHAR(100) NULL,
    LastName NVARCHAR(100) NOT NULL,
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId),
    CONSTRAINT UQ_Employee_FullName UNIQUE (FirstName, MiddleName, LastName)
);

-- Multiple NULLs allowed
INSERT INTO dbo.Employee VALUES (1, 'John', NULL, 'Smith');  -- OK
INSERT INTO dbo.Employee VALUES (2, 'John', NULL, 'Smith');  -- OK (NULL != NULL)
INSERT INTO dbo.Employee VALUES (3, 'John', 'A', 'Smith');   -- OK
INSERT INTO dbo.Employee VALUES (4, 'John', 'A', 'Smith');   -- ❌ Error: Duplicate
```

## Filtered Unique Index (Exclude NULLs)

```sql
-- Unique only among non-NULL values
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    ManagerId INT NULL,  -- Only some employees have managers
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId)
);

-- Want: Each employee can manage only one department
-- Problem: Multiple NULL ManagerIds would violate if we use Department UNIQUE

CREATE TABLE dbo.Department (
    DepartmentId INT NOT NULL,
    DepartmentName NVARCHAR(100) NOT NULL,
    ManagerId INT NULL,
    
    CONSTRAINT PK_Department PRIMARY KEY (DepartmentId)
);

-- Solution: Filtered unique index (excludes NULLs)
CREATE UNIQUE NONCLUSTERED INDEX UQ_Department_Manager
ON dbo.Department(ManagerId)
WHERE ManagerId IS NOT NULL;

-- Now only non-NULL managers must be unique
INSERT INTO dbo.Department VALUES (1, 'Engineering', NULL);  -- OK
INSERT INTO dbo.Department VALUES (2, 'Sales', NULL);        -- OK (NULL allowed)
INSERT INTO dbo.Department VALUES (3, 'Marketing', 100);     -- OK
INSERT INTO dbo.Department VALUES (4, 'Finance', 100);       -- ❌ Error: Duplicate manager
```

## Case-Sensitive Uniqueness

```sql
-- Default: Case-insensitive (depends on collation)
CREATE TABLE dbo.User (
    UserId INT NOT NULL,
    Username NVARCHAR(50) COLLATE Latin1_General_CI_AS NOT NULL,
    
    CONSTRAINT PK_User PRIMARY KEY (UserId),
    CONSTRAINT UQ_User_Username UNIQUE (Username)
);

INSERT INTO dbo.User VALUES (1, 'alice');  -- OK
INSERT INTO dbo.User VALUES (2, 'Alice');  -- ❌ Error: Considered duplicate

-- Case-sensitive uniqueness
CREATE TABLE dbo.User (
    UserId INT NOT NULL,
    Username NVARCHAR(50) COLLATE Latin1_General_CS_AS NOT NULL,
    
    CONSTRAINT PK_User PRIMARY KEY (UserId),
    CONSTRAINT UQ_User_Username UNIQUE (Username)
);

INSERT INTO dbo.User VALUES (1, 'alice');  -- OK
INSERT INTO dbo.User VALUES (2, 'Alice');  -- OK (case-sensitive)
```

## Natural Keys

```sql
-- Business natural key as alternate key
CREATE TABLE dbo.Country (
    CountryId INT NOT NULL,
    CountryCode CHAR(2) NOT NULL,   -- ISO 3166-1 alpha-2
    CountryName NVARCHAR(100) NOT NULL,
    
    CONSTRAINT PK_Country PRIMARY KEY (CountryId),
    CONSTRAINT UQ_Country_Code UNIQUE (CountryCode),
    CONSTRAINT UQ_Country_Name UNIQUE (CountryName)
);

-- Can query by natural key
SELECT * FROM dbo.Country WHERE CountryCode = 'US';
SELECT * FROM dbo.Country WHERE CountryName = 'United States';

-- Foreign keys can reference unique constraint
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    CountryCode CHAR(2) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId),
    CONSTRAINT FK_Customer_Country FOREIGN KEY (CountryCode)
        REFERENCES dbo.Country(CountryCode)
);
```

## Unique Constraints in Temporal Tables

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    ValidFrom DATETIME2(0) GENERATED ALWAYS AS ROW START NOT NULL,
    ValidTo DATETIME2(0) GENERATED ALWAYS AS ROW END NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId),
    CONSTRAINT UQ_Customer_Email UNIQUE (Email),
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.CustomerHistory));

-- Unique constraint applies to current table only
-- History table can have duplicates (historical values)
UPDATE dbo.Customer
SET Email = 'newemail@example.com'
WHERE CustomerId = 1;
-- Old email now in history table (duplicate allowed there)
```

## Adding Unique Constraints to Existing Tables

```sql
-- Check for duplicates first
SELECT Email, COUNT(*) AS DuplicateCount
FROM dbo.Customer
GROUP BY Email
HAVING COUNT(*) > 1;

-- Fix duplicates (keep first, mark others)
WITH DuplicateCTE AS (
    SELECT 
        CustomerId,
        Email,
        ROW_NUMBER() OVER (PARTITION BY Email ORDER BY CustomerId) AS RowNum
    FROM dbo.Customer
)
UPDATE dbo.Customer
SET Email = Email + '_' + CAST(RowNum AS VARCHAR)
FROM DuplicateCTE
WHERE DuplicateCTE.CustomerId = dbo.Customer.CustomerId
  AND RowNum > 1;

-- Now safe to add unique constraint
ALTER TABLE dbo.Customer
ADD CONSTRAINT UQ_Customer_Email UNIQUE (Email);
```

## Unique Constraint with Computed Columns

```sql
CREATE TABLE dbo.Person (
    PersonId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    DateOfBirth DATE NOT NULL,
    -- Computed full name
    FullName AS (FirstName + ' ' + LastName) PERSISTED,
    
    CONSTRAINT PK_Person PRIMARY KEY (PersonId),
    -- Unique on computed column
    CONSTRAINT UQ_Person_FullName_DOB UNIQUE (FullName, DateOfBirth)
);

-- Cannot have two people with same name and DOB
INSERT INTO dbo.Person VALUES (1, 'John', 'Smith', '1990-01-01');  -- OK
INSERT INTO dbo.Person VALUES (2, 'John', 'Smith', '1990-01-01');  -- ❌ Error
INSERT INTO dbo.Person VALUES (3, 'John', 'Smith', '1991-01-01');  -- OK (different DOB)
```

## Soft Delete with Unique Constraints

```sql
-- Problem: Deleted records block reuse of unique values
CREATE TABLE dbo.User (
    UserId INT NOT NULL,
    Username NVARCHAR(50) NOT NULL,
    DeletedAt DATETIME2(0) NULL,
    
    CONSTRAINT PK_User PRIMARY KEY (UserId)
);

-- Solution 1: Remove unique constraint from deleted records
-- Use filtered index to enforce uniqueness only on active records
CREATE UNIQUE NONCLUSTERED INDEX UQ_User_Username_Active
ON dbo.User(Username)
WHERE DeletedAt IS NULL;

-- Now can soft delete and reuse username
INSERT INTO dbo.User VALUES (1, 'alice', NULL);           -- Active

UPDATE dbo.User SET DeletedAt = SYSDATETIME() WHERE UserId = 1;  -- Soft delete

INSERT INTO dbo.User VALUES (2, 'alice', NULL);           -- OK (previous deleted)

-- Solution 2: Include DeletedAt in unique constraint
DROP INDEX UQ_User_Username_Active ON dbo.User;

ALTER TABLE dbo.User
ADD CONSTRAINT UQ_User_Username_DeletedAt UNIQUE (Username, DeletedAt);

-- Allows deleted duplicates (different DeletedAt), but only one active NULL
```

## Multiple Unique Constraints

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    SKU VARCHAR(50) NOT NULL,
    UPC VARCHAR(12) NULL,
    Name NVARCHAR(200) NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId),
    CONSTRAINT UQ_Product_SKU UNIQUE (SKU),
    CONSTRAINT UQ_Product_UPC UNIQUE (UPC),
    CONSTRAINT UQ_Product_Name UNIQUE (Name)
);

-- All unique values must be distinct
-- SKU, UPC, and Name are all alternate keys
```

## Viewing Unique Constraints

```sql
-- View all unique constraints
SELECT 
    tc.CONSTRAINT_NAME,
    tc.TABLE_NAME,
    kcu.COLUMN_NAME,
    tc.CONSTRAINT_TYPE
FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS tc
INNER JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE kcu
    ON tc.CONSTRAINT_NAME = kcu.CONSTRAINT_NAME
WHERE tc.CONSTRAINT_TYPE = 'UNIQUE'
ORDER BY tc.TABLE_NAME, tc.CONSTRAINT_NAME, kcu.ORDINAL_POSITION;

-- View unique constraints with multiple columns
SELECT 
    tc.TABLE_NAME,
    tc.CONSTRAINT_NAME,
    STRING_AGG(kcu.COLUMN_NAME, ', ') WITHIN GROUP (ORDER BY kcu.ORDINAL_POSITION) AS Columns
FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS tc
INNER JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE kcu
    ON tc.CONSTRAINT_NAME = kcu.CONSTRAINT_NAME
WHERE tc.CONSTRAINT_TYPE = 'UNIQUE'
GROUP BY tc.TABLE_NAME, tc.CONSTRAINT_NAME
ORDER BY tc.TABLE_NAME;
```

## Error Handling

```sql
CREATE PROCEDURE dbo.usp_RegisterUser
    @Username NVARCHAR(50),
    @Email NVARCHAR(255)
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        INSERT INTO dbo.User (Username, Email)
        VALUES (@Username, @Email);
        
        SELECT 'SUCCESS' AS Result;
    END TRY
    BEGIN CATCH
        IF ERROR_NUMBER() = 2627  -- Unique constraint violation
        BEGIN
            DECLARE @ErrorMsg NVARCHAR(4000) = ERROR_MESSAGE();
            
            IF @ErrorMsg LIKE '%UQ_User_Username%'
            BEGIN
                RAISERROR('Username "%s" is already taken', 16, 1, @Username);
            END
            ELSE IF @ErrorMsg LIKE '%UQ_User_Email%'
            BEGIN
                RAISERROR('Email "%s" is already registered', 16, 1, @Email);
            END
            ELSE
            BEGIN
                THROW;
            END;
        END
        ELSE
        BEGIN
            THROW;
        END;
    END CATCH;
END;
```

## Performance Considerations

```sql
-- Unique constraints create indexes automatically
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId),
    CONSTRAINT UQ_Customer_Email UNIQUE (Email)  -- Creates index on Email
);

-- Queries benefit from index
SELECT * FROM dbo.Customer
WHERE Email = 'alice@example.com';  -- Index seek (fast)

-- Can specify index options
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId),
    CONSTRAINT UQ_Customer_Email UNIQUE (Email)
    WITH (FILLFACTOR = 80)
);
```

## Why Duplicates Are a Problem

1. **Business rule violations**: Two accounts with same email
2. **Data quality issues**: Cannot identify correct record
3. **Application logic errors**: Unexpected multiple results
4. **User experience**: Confusing duplicate usernames/emails
5. **Integration problems**: External systems expect unique identifiers

## Symptoms

- Comments like "email should be unique"
- Application code checking for duplicates before insert
- Support tickets about "duplicate accounts"
- Deduplication scripts running regularly

## Benefits

- **Data integrity**: Impossible to insert duplicates
- **Natural key support**: Can query by business identifiers
- **Self-documenting**: Schema shows which values must be unique
- **Automatic indexing**: Improved query performance
- **Foreign key targets**: Other tables can reference unique columns

## Trade-offs

- **Insert failures**: Must handle unique violations
- **Soft delete complexity**: Deleted records block reuse (use filtered indexes)
- **Index overhead**: Each unique constraint adds an index
- **NULL handling**: Multiple NULLs allowed (might not be desired)

## When to Use

✅ Use unique constraints for:
- Natural business keys (email, username, SSN)
- External identifiers (SKU, order number)
- Codes and abbreviations (country code, currency code)
- Preventing logical duplicates

❌ Consider alternatives for:
- When duplicates are valid (names, descriptions)
- Temporary or staging data
- When soft deletes need reusable values (use filtered indexes)

## See Also

- [Schema Constraints](./schema-constraints.md)
- [Check Constraints](./check-constraints.md)
- [Foreign Key Constraints](./foreign-key-constraints.md)
- [Named Constraints](./named-constraints.md)
- [Soft Deletes](./soft-deletes.md)
