# Foreign Key Constraints

> Enforce referential integrity between tables—prevent orphaned records and maintain data consistency.

## Problem

Without foreign key constraints, relationships between tables can break. Child records reference parent records that don't exist, leading to data inconsistencies and application errors.

## Example

### ❌ Before (No Referential Integrity)

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,  -- No FK constraint!
    Total DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId)
);

-- Insert order for non-existent customer
INSERT INTO dbo.Order (OrderId, CustomerId, Total)
VALUES (1, 999, 100.00);
-- ❌ Succeeds! Creates orphaned order

-- Delete customer with orders
DELETE FROM dbo.Customer WHERE CustomerId = 123;
-- ❌ Succeeds! Orphans all orders for this customer
```

### ✅ After (Foreign Key Enforces Integrity)

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerId)
        REFERENCES dbo.Customer(CustomerId)
);

-- Try to insert order for non-existent customer
INSERT INTO dbo.Order (OrderId, CustomerId, Total)
VALUES (1, 999, 100.00);
-- ❌ Error: The INSERT statement conflicted with the FOREIGN KEY constraint "FK_Order_Customer"

-- Try to delete customer with orders
DELETE FROM dbo.Customer WHERE CustomerId = 123;
-- ❌ Error: The DELETE statement conflicted with the REFERENCE constraint "FK_Order_Customer"
```

## Basic Foreign Key

```sql
CREATE TABLE dbo.OrderLineItem (
    OrderLineItemId INT NOT NULL,
    OrderId INT NOT NULL,
    ProductId INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_OrderLineItem PRIMARY KEY (OrderLineItemId),
    CONSTRAINT FK_OrderLineItem_Order FOREIGN KEY (OrderId)
        REFERENCES dbo.Order(OrderId),
    CONSTRAINT FK_OrderLineItem_Product FOREIGN KEY (ProductId)
        REFERENCES dbo.Product(ProductId)
);
```

## Composite Foreign Keys

```sql
CREATE TABLE dbo.Store (
    StoreId INT NOT NULL,
    RegionId INT NOT NULL,
    StoreName NVARCHAR(200) NOT NULL,
    
    CONSTRAINT PK_Store PRIMARY KEY (StoreId, RegionId)
);

CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    StoreId INT NOT NULL,
    RegionId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId),
    -- Composite FK matches composite PK
    CONSTRAINT FK_Employee_Store FOREIGN KEY (StoreId, RegionId)
        REFERENCES dbo.Store(StoreId, RegionId)
);
```

## CASCADE Options

### ON DELETE CASCADE

Automatically delete child records when parent is deleted:

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerId)
        REFERENCES dbo.Customer(CustomerId)
        ON DELETE CASCADE  -- Delete orders when customer is deleted
);

CREATE TABLE dbo.OrderLineItem (
    OrderLineItemId INT NOT NULL,
    OrderId INT NOT NULL,
    
    CONSTRAINT PK_OrderLineItem PRIMARY KEY (OrderLineItemId),
    CONSTRAINT FK_OrderLineItem_Order FOREIGN KEY (OrderId)
        REFERENCES dbo.Order(OrderId)
        ON DELETE CASCADE  -- Delete line items when order is deleted
);

-- Delete customer
DELETE FROM dbo.Customer WHERE CustomerId = 123;
-- ✅ Cascade deletes: Orders deleted, then OrderLineItems deleted
```

### ON DELETE SET NULL

Set foreign key to NULL when parent is deleted:

```sql
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    ManagerId INT NULL,  -- Must be nullable
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId),
    CONSTRAINT FK_Employee_Manager FOREIGN KEY (ManagerId)
        REFERENCES dbo.Employee(EmployeeId)
        ON DELETE SET NULL  -- Remove manager reference when manager deleted
);

-- Delete manager
DELETE FROM dbo.Employee WHERE EmployeeId = 5;
-- ✅ All employees with ManagerId = 5 now have ManagerId = NULL
```

### ON DELETE SET DEFAULT

Set foreign key to default value when parent is deleted:

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    CategoryId INT NOT NULL DEFAULT 1,  -- Default category
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId),
    CONSTRAINT FK_Product_Category FOREIGN KEY (CategoryId)
        REFERENCES dbo.Category(CategoryId)
        ON DELETE SET DEFAULT  -- Use default category when category deleted
);

-- Must have default category
INSERT INTO dbo.Category (CategoryId, Name)
VALUES (1, 'Uncategorized');

-- Delete category
DELETE FROM dbo.Category WHERE CategoryId = 5;
-- ✅ All products in category 5 moved to category 1
```

### ON UPDATE CASCADE

Update foreign key when parent key changes:

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerId)
        REFERENCES dbo.Customer(CustomerId)
        ON UPDATE CASCADE  -- Update orders when customer ID changes
);

-- Change customer ID
UPDATE dbo.Customer
SET CustomerId = 999
WHERE CustomerId = 123;
-- ✅ All orders for customer 123 now reference customer 999
```

## Self-Referencing Foreign Keys

```sql
-- Organizational hierarchy
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    ManagerId INT NULL,
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId),
    CONSTRAINT FK_Employee_Manager FOREIGN KEY (ManagerId)
        REFERENCES dbo.Employee(EmployeeId)
);

-- CEO has no manager
INSERT INTO dbo.Employee (EmployeeId, FirstName, LastName, ManagerId)
VALUES (1, 'Jane', 'CEO', NULL);

-- VP reports to CEO
INSERT INTO dbo.Employee (EmployeeId, FirstName, LastName, ManagerId)
VALUES (2, 'John', 'VP', 1);
```

## Multiple Foreign Keys to Same Table

```sql
CREATE TABLE dbo.Payment (
    PaymentId INT NOT NULL,
    OrderId INT NOT NULL,
    ApprovedBy INT NOT NULL,  -- Employee who approved
    ProcessedBy INT NOT NULL, -- Employee who processed
    
    CONSTRAINT PK_Payment PRIMARY KEY (PaymentId),
    CONSTRAINT FK_Payment_Order FOREIGN KEY (OrderId)
        REFERENCES dbo.Order(OrderId),
    CONSTRAINT FK_Payment_ApprovedBy FOREIGN KEY (ApprovedBy)
        REFERENCES dbo.Employee(EmployeeId),
    CONSTRAINT FK_Payment_ProcessedBy FOREIGN KEY (ProcessedBy)
        REFERENCES dbo.Employee(EmployeeId)
);
```

## Adding Foreign Keys to Existing Tables

```sql
-- Add FK to existing table
ALTER TABLE dbo.Order
ADD CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerId)
    REFERENCES dbo.Customer(CustomerId);

-- Add FK with CASCADE
ALTER TABLE dbo.OrderLineItem
ADD CONSTRAINT FK_OrderLineItem_Order FOREIGN KEY (OrderId)
    REFERENCES dbo.Order(OrderId)
    ON DELETE CASCADE;

-- Find orphaned records before adding FK
SELECT o.OrderId, o.CustomerId
FROM dbo.Order o
LEFT JOIN dbo.Customer c ON o.CustomerId = c.CustomerId
WHERE c.CustomerId IS NULL;

-- Fix orphaned records
DELETE FROM dbo.Order
WHERE CustomerId NOT IN (SELECT CustomerId FROM dbo.Customer);

-- Now safe to add FK
ALTER TABLE dbo.Order
ADD CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerId)
    REFERENCES dbo.Customer(CustomerId);
```

## Disabling and Enabling Constraints

```sql
-- Disable FK for bulk load
ALTER TABLE dbo.Order NOCHECK CONSTRAINT FK_Order_Customer;

-- Bulk insert (skips FK validation)
INSERT INTO dbo.Order (OrderId, CustomerId, Total)
SELECT OrderId, CustomerId, Total
FROM ExternalDataSource;

-- Re-enable and validate existing data
ALTER TABLE dbo.Order CHECK CONSTRAINT FK_Order_Customer;

-- Re-enable without validation (leaves orphans!)
ALTER TABLE dbo.Order WITH NOCHECK CHECK CONSTRAINT FK_Order_Customer;
```

## Handling Circular References

```sql
-- Problem: Table A references B, B references A
CREATE TABLE dbo.Department (
    DepartmentId INT NOT NULL,
    DepartmentName NVARCHAR(100) NOT NULL,
    ManagerId INT NULL,  -- References Employee
    
    CONSTRAINT PK_Department PRIMARY KEY (DepartmentId)
);

CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    DepartmentId INT NOT NULL,  -- References Department
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId)
);

-- Solution: Add FKs after both tables exist
ALTER TABLE dbo.Department
ADD CONSTRAINT FK_Department_Manager FOREIGN KEY (ManagerId)
    REFERENCES dbo.Employee(EmployeeId);

ALTER TABLE dbo.Employee
ADD CONSTRAINT FK_Employee_Department FOREIGN KEY (DepartmentId)
    REFERENCES dbo.Department(DepartmentId);

-- Insert requires careful ordering or disabling constraints temporarily
ALTER TABLE dbo.Department NOCHECK CONSTRAINT FK_Department_Manager;

INSERT INTO dbo.Department (DepartmentId, DepartmentName, ManagerId)
VALUES (1, 'Engineering', NULL);

INSERT INTO dbo.Employee (EmployeeId, FirstName, DepartmentId)
VALUES (100, 'Jane', 1);

UPDATE dbo.Department
SET ManagerId = 100
WHERE DepartmentId = 1;

ALTER TABLE dbo.Department CHECK CONSTRAINT FK_Department_Manager;
```

## Foreign Keys with Unique Constraints

```sql
-- One-to-one relationship
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId)
);

CREATE TABLE dbo.EmployeeDetails (
    EmployeeDetailsId INT NOT NULL,
    EmployeeId INT NOT NULL,  -- One-to-one with Employee
    DateOfBirth DATE NOT NULL,
    SSN VARCHAR(11) NOT NULL,
    
    CONSTRAINT PK_EmployeeDetails PRIMARY KEY (EmployeeDetailsId),
    CONSTRAINT UQ_EmployeeDetails_EmployeeId UNIQUE (EmployeeId),
    CONSTRAINT FK_EmployeeDetails_Employee FOREIGN KEY (EmployeeId)
        REFERENCES dbo.Employee(EmployeeId)
        ON DELETE CASCADE
);
-- Each employee can have only one EmployeeDetails record
```

## Indexed Foreign Keys

```sql
-- Foreign keys are not automatically indexed
-- Add indexes for join performance
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerId)
        REFERENCES dbo.Customer(CustomerId)
);

-- Create index on FK column for faster joins
CREATE NONCLUSTERED INDEX IX_Order_CustomerId
ON dbo.Order(CustomerId);

-- Query benefits from index
SELECT c.Name, COUNT(*) AS OrderCount
FROM dbo.Customer c
INNER JOIN dbo.Order o ON c.CustomerId = o.CustomerId
GROUP BY c.Name;
```

## Viewing Foreign Keys

```sql
-- View all foreign keys in database
SELECT 
    fk.name AS ForeignKeyName,
    tp.name AS ParentTable,
    cp.name AS ParentColumn,
    tr.name AS ReferencedTable,
    cr.name AS ReferencedColumn,
    fk.delete_referential_action_desc AS DeleteAction,
    fk.update_referential_action_desc AS UpdateAction
FROM sys.foreign_keys fk
INNER JOIN sys.foreign_key_columns fkc ON fk.object_id = fkc.constraint_object_id
INNER JOIN sys.tables tp ON fkc.parent_object_id = tp.object_id
INNER JOIN sys.columns cp ON fkc.parent_object_id = cp.object_id 
    AND fkc.parent_column_id = cp.column_id
INNER JOIN sys.tables tr ON fkc.referenced_object_id = tr.object_id
INNER JOIN sys.columns cr ON fkc.referenced_object_id = cr.object_id 
    AND fkc.referenced_column_id = cr.column_id
ORDER BY tp.name, fk.name;

-- Find tables without foreign keys
SELECT t.name AS TableName
FROM sys.tables t
WHERE NOT EXISTS (
    SELECT 1
    FROM sys.foreign_keys fk
    WHERE fk.parent_object_id = t.object_id
)
AND t.name NOT LIKE 'sys%'
ORDER BY t.name;
```

## Error Handling

```sql
CREATE PROCEDURE dbo.usp_CreateOrder
    @OrderId INT,
    @CustomerId INT,
    @Total DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        INSERT INTO dbo.Order (OrderId, CustomerId, Total)
        VALUES (@OrderId, @CustomerId, @Total);
        
        SELECT 'SUCCESS' AS Result;
    END TRY
    BEGIN CATCH
        IF ERROR_NUMBER() = 547  -- FK or CHECK constraint violation
        BEGIN
            IF ERROR_MESSAGE() LIKE '%FK_Order_Customer%'
            BEGIN
                RAISERROR('Customer %d does not exist', 16, 1, @CustomerId);
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

## Soft Delete with Foreign Keys

```sql
-- When using soft deletes, use filtered indexes
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    DeletedAt DATETIME2(0) NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    DeletedAt DATETIME2(0) NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    -- FK still enforces integrity
    CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerId)
        REFERENCES dbo.Customer(CustomerId)
);

-- Cannot hard delete customer with orders
-- Must soft delete both
UPDATE dbo.Customer
SET DeletedAt = SYSDATETIME()
WHERE CustomerId = 123;

UPDATE dbo.Order
SET DeletedAt = SYSDATETIME()
WHERE CustomerId = 123;
```

## Why Missing Foreign Keys Are a Problem

1. **Orphaned records**: Child records reference non-existent parents
2. **Data corruption**: Joins return incomplete or incorrect results
3. **Application errors**: Null reference exceptions from missing data
4. **Query complexity**: Must check existence before operations
5. **No cascade control**: Manual cleanup of related records

## Symptoms

- Queries with LEFT JOIN finding NULL parent records
- Application errors about "record not found"
- Data cleanup scripts to remove orphans
- Comments warning "customer must exist"

## Benefits

- **Data integrity**: Impossible to create orphaned records
- **Automatic validation**: Database prevents invalid references
- **Cascade operations**: Automatic handling of deletes/updates
- **Self-documenting**: Schema shows relationships clearly
- **Query optimizer**: Better execution plans with FK metadata

## Trade-offs

- **Insert ordering**: Must insert parent before child
- **Delete restrictions**: Cannot delete parent with children (without CASCADE)
- **Performance overhead**: Validation on every insert/update (minimal)
- **Circular references**: Requires careful handling

## When to Use

✅ Always use foreign keys for:
- Parent-child relationships
- Lookup/reference tables
- Many-to-many join tables
- Self-referencing hierarchies

❌ Consider alternatives for:
- External system references (IDs not in database)
- Temporal data (references to deleted records)
- Denormalized reporting tables

## See Also

- [Schema Constraints](./schema-constraints.md)
- [Check Constraints](./check-constraints.md)
- [Unique Constraints](./unique-constraints.md)
- [Named Constraints](./named-constraints.md)
- [Soft Deletes](./soft-deletes.md)
