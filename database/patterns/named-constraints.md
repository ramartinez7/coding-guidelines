# Named Constraints

> Use explicit, descriptive names for all constraints—make schema self-documenting and errors understandable.

## Problem

SQL Server generates cryptic system names for anonymous constraints. When constraints are violated or need to be modified, these auto-generated names are difficult to understand and maintain.

## Example

### ❌ Before (Anonymous Constraints)

```sql
CREATE TABLE dbo.Product (
    ProductId INT PRIMARY KEY,
    Name NVARCHAR(200) NOT NULL,
    Price DECIMAL(18,2) CHECK (Price >= 0),
    CategoryId INT FOREIGN KEY REFERENCES dbo.Category(CategoryId)
);

-- Error messages are cryptic
INSERT INTO dbo.Product (ProductId, Name, Price, CategoryId)
VALUES (1, 'Widget', -10.00, 1);
-- ❌ Error: The INSERT statement conflicted with the CHECK constraint "CK__Product__Price__5EBF139D".

-- Finding constraints is difficult
SELECT name FROM sys.check_constraints
WHERE parent_object_id = OBJECT_ID('dbo.Product');
-- Returns: "CK__Product__Price__5EBF139D"  (what does this check?)
```

### ✅ After (Named Constraints)

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    CategoryId INT NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId),
    CONSTRAINT CK_Product_Price CHECK (Price >= 0),
    CONSTRAINT FK_Product_Category FOREIGN KEY (CategoryId)
        REFERENCES dbo.Category(CategoryId)
);

-- Error messages are clear
INSERT INTO dbo.Product (ProductId, Name, Price, CategoryId)
VALUES (1, 'Widget', -10.00, 1);
-- ❌ Error: The INSERT statement conflicted with the CHECK constraint "CK_Product_Price".

-- Finding constraints is easy
SELECT name FROM sys.check_constraints
WHERE parent_object_id = OBJECT_ID('dbo.Product');
-- Returns: "CK_Product_Price"  (clearly checking price)
```

## Naming Conventions

### Primary Key Constraints

```sql
-- Convention: PK_{TableName}
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

-- Composite primary key
CREATE TABLE dbo.OrderLineItem (
    OrderId INT NOT NULL,
    LineItemId INT NOT NULL,
    ProductId INT NOT NULL,
    
    CONSTRAINT PK_OrderLineItem PRIMARY KEY (OrderId, LineItemId)
);
```

### Foreign Key Constraints

```sql
-- Convention: FK_{ChildTable}_{ParentTable}
-- Alternative: FK_{ChildTable}_{ColumnName}
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    ShippingAddressId INT NULL,
    BillingAddressId INT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerId)
        REFERENCES dbo.Customer(CustomerId),
    -- Disambiguate multiple FKs to same table
    CONSTRAINT FK_Order_ShippingAddress FOREIGN KEY (ShippingAddressId)
        REFERENCES dbo.Address(AddressId),
    CONSTRAINT FK_Order_BillingAddress FOREIGN KEY (BillingAddressId)
        REFERENCES dbo.Address(AddressId)
);
```

### Unique Constraints

```sql
-- Convention: UQ_{TableName}_{ColumnName}
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    EmployeeNumber VARCHAR(20) NOT NULL,
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId),
    CONSTRAINT UQ_Employee_Email UNIQUE (Email),
    CONSTRAINT UQ_Employee_EmployeeNumber UNIQUE (EmployeeNumber)
);

-- Composite unique constraint
CREATE TABLE dbo.Student (
    StudentId INT NOT NULL,
    SchoolId INT NOT NULL,
    StudentNumber VARCHAR(20) NOT NULL,
    
    CONSTRAINT PK_Student PRIMARY KEY (StudentId),
    CONSTRAINT UQ_Student_SchoolNumber UNIQUE (SchoolId, StudentNumber)
);
```

### Check Constraints

```sql
-- Convention: CK_{TableName}_{ColumnName}
-- Alternative: CK_{TableName}_{BusinessRule}
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    DiscountPercent DECIMAL(5,2) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId),
    CONSTRAINT CK_Product_Price CHECK (Price >= 0),
    CONSTRAINT CK_Product_DiscountPercent CHECK (DiscountPercent >= 0 AND DiscountPercent <= 100),
    CONSTRAINT CK_Product_Status CHECK (Status IN ('Active', 'Discontinued', 'OutOfStock'))
);

-- Multi-column check constraint
CREATE TABLE dbo.Event (
    EventId INT NOT NULL,
    StartDate DATETIME2(0) NOT NULL,
    EndDate DATETIME2(0) NOT NULL,
    
    CONSTRAINT PK_Event PRIMARY KEY (EventId),
    CONSTRAINT CK_Event_DateRange CHECK (EndDate > StartDate)
);
```

### Default Constraints

```sql
-- Convention: DF_{TableName}_{ColumnName}
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    OrderDate DATETIME2(0) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT DF_Order_OrderDate DEFAULT SYSDATETIME() FOR OrderDate,
    CONSTRAINT DF_Order_Status DEFAULT 'Pending' FOR Status
);
```

### Index Names

```sql
-- Convention for unique indexes: UQ_{TableName}_{ColumnName}
-- Convention for non-unique: IX_{TableName}_{ColumnName}
-- Include optional descriptor: IX_{TableName}_{ColumnName}_{Descriptor}

CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    OrderDate DATETIME2(0) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId)
);

-- Non-unique index
CREATE NONCLUSTERED INDEX IX_Order_CustomerId
ON dbo.Order(CustomerId);

-- Covering index with included columns
CREATE NONCLUSTERED INDEX IX_Order_CustomerId_Covering
ON dbo.Order(CustomerId)
INCLUDE (OrderDate, Status);

-- Filtered index
CREATE NONCLUSTERED INDEX IX_Order_Status_Active
ON dbo.Order(Status)
WHERE Status = 'Active';
```

## Benefits of Named Constraints

### Clear Error Messages

```sql
-- Named constraint error
INSERT INTO dbo.Product (ProductId, Name, Price)
VALUES (1, 'Widget', -10.00);
-- Error: The INSERT statement conflicted with the CHECK constraint "CK_Product_Price".
-- ✅ Immediately know: Price constraint on Product table

-- Anonymous constraint error
-- Error: The INSERT statement conflicted with the CHECK constraint "CK__Product__Price__5EBF139D".
-- ❌ Unclear what this checks without looking up constraint definition
```

### Easier Maintenance

```sql
-- Named: Easy to drop and recreate
ALTER TABLE dbo.Product DROP CONSTRAINT CK_Product_Price;
ALTER TABLE dbo.Product ADD CONSTRAINT CK_Product_Price CHECK (Price >= 0);

-- Anonymous: Must find system-generated name first
SELECT name FROM sys.check_constraints
WHERE parent_object_id = OBJECT_ID('dbo.Product')
  AND definition LIKE '%Price%';
-- Then use the cryptic name
ALTER TABLE dbo.Product DROP CONSTRAINT CK__Product__Price__5EBF139D;
```

### Self-Documenting Schema

```sql
-- Named constraints reveal intent
SELECT 
    t.name AS TableName,
    c.name AS ConstraintName,
    c.type_desc AS ConstraintType
FROM sys.tables t
INNER JOIN sys.objects c ON t.object_id = c.parent_object_id
WHERE c.type IN ('PK', 'F', 'UQ', 'C', 'D')
  AND t.name = 'Order'
ORDER BY ConstraintType, ConstraintName;

-- Results show clear purpose:
-- PK_Order               PRIMARY_KEY_CONSTRAINT
-- FK_Order_Customer      FOREIGN_KEY_CONSTRAINT
-- UQ_Order_OrderNumber   UNIQUE_CONSTRAINT
-- CK_Order_Status        CHECK_CONSTRAINT
-- DF_Order_OrderDate     DEFAULT_CONSTRAINT
```

## Naming Patterns for Complex Scenarios

### Self-Referencing Foreign Keys

```sql
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    ManagerId INT NULL,
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId),
    -- Include "Self" or role name to indicate self-reference
    CONSTRAINT FK_Employee_Manager FOREIGN KEY (ManagerId)
        REFERENCES dbo.Employee(EmployeeId)
);

-- Alternative with clearer role indication
CONSTRAINT FK_Employee_Manager_Self FOREIGN KEY (ManagerId)
    REFERENCES dbo.Employee(EmployeeId)
```

### Multiple Constraints on Same Column

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    DiscountPrice DECIMAL(18,2) NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId),
    CONSTRAINT CK_Product_Price_NonNegative CHECK (Price >= 0),
    CONSTRAINT CK_Product_DiscountPrice_NonNegative CHECK (DiscountPrice IS NULL OR DiscountPrice >= 0),
    CONSTRAINT CK_Product_DiscountPrice_LessThanPrice CHECK (DiscountPrice IS NULL OR DiscountPrice < Price)
);
```

### Temporal Constraints

```sql
CREATE TABLE dbo.Subscription (
    SubscriptionId INT NOT NULL,
    CustomerId INT NOT NULL,
    StartDate DATE NOT NULL,
    EndDate DATE NULL,
    
    CONSTRAINT PK_Subscription PRIMARY KEY (SubscriptionId),
    CONSTRAINT CK_Subscription_EndDate_AfterStart CHECK (EndDate IS NULL OR EndDate >= StartDate),
    CONSTRAINT DF_Subscription_StartDate DEFAULT CAST(SYSDATETIME() AS DATE) FOR StartDate
);
```

## Refactoring Anonymous Constraints

```sql
-- Find anonymous constraints
SELECT 
    t.name AS TableName,
    c.name AS ConstraintName,
    c.type_desc AS ConstraintType
FROM sys.tables t
INNER JOIN sys.objects c ON t.object_id = c.parent_object_id
WHERE c.type IN ('PK', 'F', 'UQ', 'C', 'D')
  AND c.is_ms_shipped = 0
  AND (
    c.name LIKE '%__[0-9][0-9][0-9][0-9]%' OR  -- System-generated pattern
    c.name NOT LIKE 'PK_%' AND c.type = 'PK' OR
    c.name NOT LIKE 'FK_%' AND c.type = 'F' OR
    c.name NOT LIKE 'UQ_%' AND c.type = 'UQ' OR
    c.name NOT LIKE 'CK_%' AND c.type = 'C' OR
    c.name NOT LIKE 'DF_%' AND c.type = 'D'
  )
ORDER BY t.name, c.type_desc, c.name;

-- Script to rename CHECK constraint
DECLARE @OldName NVARCHAR(128) = 'CK__Product__Price__5EBF139D';
DECLARE @TableName NVARCHAR(128) = 'Product';

-- Get constraint definition
DECLARE @Definition NVARCHAR(MAX);
SELECT @Definition = definition
FROM sys.check_constraints
WHERE name = @OldName;

-- Drop old constraint
EXEC('ALTER TABLE dbo.' + @TableName + ' DROP CONSTRAINT ' + @OldName);

-- Create with proper name
EXEC('ALTER TABLE dbo.' + @TableName + ' ADD CONSTRAINT CK_Product_Price CHECK ' + @Definition);
```

## Documentation Generation

```sql
-- Generate constraint documentation
SELECT 
    SCHEMA_NAME(t.schema_id) + '.' + t.name AS TableName,
    c.name AS ConstraintName,
    c.type_desc AS ConstraintType,
    CASE c.type
        WHEN 'PK' THEN 'Primary key on ' + STRING_AGG(col.name, ', ')
        WHEN 'UQ' THEN 'Unique constraint on ' + STRING_AGG(col.name, ', ')
        WHEN 'F' THEN 'Foreign key to ' + 
            SCHEMA_NAME(ref_t.schema_id) + '.' + ref_t.name
        WHEN 'C' THEN 'Check: ' + cc.definition
        WHEN 'D' THEN 'Default: ' + dc.definition
    END AS Description
FROM sys.tables t
INNER JOIN sys.objects c ON t.object_id = c.parent_object_id
LEFT JOIN sys.index_columns ic ON c.object_id = ic.object_id
LEFT JOIN sys.columns col ON ic.object_id = col.object_id AND ic.column_id = col.column_id
LEFT JOIN sys.check_constraints cc ON c.object_id = cc.object_id
LEFT JOIN sys.default_constraints dc ON c.object_id = dc.object_id
LEFT JOIN sys.foreign_keys fk ON c.object_id = fk.object_id
LEFT JOIN sys.tables ref_t ON fk.referenced_object_id = ref_t.object_id
WHERE c.type IN ('PK', 'F', 'UQ', 'C', 'D')
  AND t.is_ms_shipped = 0
GROUP BY 
    SCHEMA_NAME(t.schema_id), t.name, c.name, c.type_desc, c.type,
    cc.definition, dc.definition, SCHEMA_NAME(ref_t.schema_id), ref_t.name
ORDER BY TableName, ConstraintType, ConstraintName;
```

## Code Generation from Named Constraints

```sql
-- Generate ALTER TABLE statements for all constraints
SELECT 
    'ALTER TABLE ' + SCHEMA_NAME(t.schema_id) + '.' + t.name +
    ' ADD CONSTRAINT ' + c.name + 
    CASE c.type
        WHEN 'PK' THEN ' PRIMARY KEY (' + STRING_AGG(col.name, ', ') + ')'
        WHEN 'UQ' THEN ' UNIQUE (' + STRING_AGG(col.name, ', ') + ')'
        WHEN 'C' THEN ' CHECK ' + cc.definition
        WHEN 'D' THEN ' DEFAULT ' + dc.definition + ' FOR ' + col.name
        WHEN 'F' THEN ' FOREIGN KEY (' + STRING_AGG(col.name, ', ') + ') REFERENCES ' +
            SCHEMA_NAME(ref_t.schema_id) + '.' + ref_t.name + '(' + 
            STRING_AGG(ref_col.name, ', ') + ')'
    END + ';' AS CreateStatement
FROM sys.tables t
INNER JOIN sys.objects c ON t.object_id = c.parent_object_id
LEFT JOIN sys.index_columns ic ON c.object_id = ic.object_id
LEFT JOIN sys.columns col ON ic.object_id = col.object_id AND ic.column_id = col.column_id
LEFT JOIN sys.check_constraints cc ON c.object_id = cc.object_id
LEFT JOIN sys.default_constraints dc ON c.object_id = dc.object_id
LEFT JOIN sys.foreign_keys fk ON c.object_id = fk.object_id
LEFT JOIN sys.foreign_key_columns fkc ON fk.object_id = fkc.constraint_object_id
LEFT JOIN sys.tables ref_t ON fk.referenced_object_id = ref_t.object_id
LEFT JOIN sys.columns ref_col ON fkc.referenced_object_id = ref_col.object_id 
    AND fkc.referenced_column_id = ref_col.column_id
WHERE c.type IN ('PK', 'F', 'UQ', 'C', 'D')
  AND t.name = 'YourTableName'
GROUP BY 
    SCHEMA_NAME(t.schema_id), t.name, c.name, c.type,
    cc.definition, dc.definition, col.name,
    SCHEMA_NAME(ref_t.schema_id), ref_t.name
ORDER BY c.type;
```

## Why Anonymous Constraints Are a Problem

1. **Unclear errors**: Cryptic constraint names in error messages
2. **Hard to maintain**: Must look up system-generated names
3. **Poor documentation**: Schema doesn't reveal intent
4. **Refactoring difficulty**: Cannot predict constraint names
5. **Script portability**: Names differ across environments

## Symptoms

- Error messages with unreadable constraint names
- Queries to find "which constraint is this?"
- Copy-paste errors from system-generated names
- Difficulty explaining constraints to team

## Benefits

- **Readable errors**: Clear constraint violations
- **Self-documenting**: Names explain purpose
- **Easy maintenance**: Predictable naming for scripts
- **Team understanding**: Junior developers understand schema
- **Consistent naming**: Same convention across database

## Trade-offs

- **More verbose**: Longer CREATE TABLE statements
- **Naming decisions**: Must decide on conventions
- **Initial effort**: More typing upfront

## When to Use

✅ Always use named constraints for:
- All production tables
- Shared/team databases
- Long-lived applications
- Any schema others will maintain

❌ Might skip for:
- Temporary tables in scripts
- Personal scratch work
- Throwaway prototypes

## See Also

- [Schema Constraints](./schema-constraints.md)
- [Check Constraints](./check-constraints.md)
- [Foreign Key Constraints](./foreign-key-constraints.md)
- [Unique Constraints](./unique-constraints.md)
