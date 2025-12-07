# Check Constraints

> Use CHECK constraints to enforce domain rules and value ranges—prevent invalid data at the database level.

## Problem

Application code validates business rules, but those rules can be bypassed by bugs, rogue queries, or data imports. Invalid data enters the database and corrupts the domain model.

## Example

### ❌ Before (Validation Only in Application)

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    DiscountPercent DECIMAL(5,2) NOT NULL,
    StockQuantity INT NOT NULL,
    Status VARCHAR(20) NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId)
);

-- Nothing prevents invalid data
INSERT INTO dbo.Product (ProductId, Name, Price, DiscountPercent, StockQuantity, Status)
VALUES (1, 'Widget', -10.00, 150, -5, 'InvalidStatus');
-- ❌ Succeeds! Negative price, invalid discount, negative stock, bad status
```

### ✅ After (Check Constraints Enforce Rules)

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    DiscountPercent DECIMAL(5,2) NOT NULL,
    StockQuantity INT NOT NULL,
    Status VARCHAR(20) NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId),
    CONSTRAINT CK_Product_Price CHECK (Price >= 0),
    CONSTRAINT CK_Product_DiscountPercent CHECK (DiscountPercent >= 0 AND DiscountPercent <= 100),
    CONSTRAINT CK_Product_StockQuantity CHECK (StockQuantity >= 0),
    CONSTRAINT CK_Product_Status CHECK (Status IN ('Active', 'Discontinued', 'OutOfStock', 'PreOrder'))
);

-- Now invalid data is impossible
INSERT INTO dbo.Product (ProductId, Name, Price, DiscountPercent, StockQuantity, Status)
VALUES (1, 'Widget', -10.00, 150, -5, 'InvalidStatus');
-- ❌ Error: The INSERT statement conflicted with the CHECK constraint "CK_Product_Price"
```

## Simple Range Constraints

```sql
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    Age INT NOT NULL,
    Salary DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId),
    CONSTRAINT CK_Employee_Age CHECK (Age >= 18 AND Age <= 100),
    CONSTRAINT CK_Employee_Salary CHECK (Salary > 0)
);
```

## Enumeration Constraints

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    Status VARCHAR(20) NOT NULL,
    Priority VARCHAR(10) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT CK_Order_Status CHECK (
        Status IN ('Pending', 'Processing', 'Shipped', 'Delivered', 'Cancelled')
    ),
    CONSTRAINT CK_Order_Priority CHECK (
        Priority IN ('Low', 'Normal', 'High', 'Urgent')
    )
);
```

## Multi-Column Constraints

```sql
CREATE TABLE dbo.Event (
    EventId INT NOT NULL,
    EventName NVARCHAR(200) NOT NULL,
    StartDate DATETIME2(0) NOT NULL,
    EndDate DATETIME2(0) NOT NULL,
    MinAttendees INT NOT NULL,
    MaxAttendees INT NOT NULL,
    
    CONSTRAINT PK_Event PRIMARY KEY (EventId),
    CONSTRAINT CK_Event_DateRange CHECK (EndDate > StartDate),
    CONSTRAINT CK_Event_AttendeeRange CHECK (MaxAttendees >= MinAttendees),
    CONSTRAINT CK_Event_MinAttendees CHECK (MinAttendees >= 0)
);
```

## Pattern Matching Constraints

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    PhoneNumber VARCHAR(20) NULL,
    PostalCode VARCHAR(10) NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId),
    -- Simple email validation
    CONSTRAINT CK_Customer_Email CHECK (Email LIKE '%_@__%.__%'),
    -- Phone format: (XXX) XXX-XXXX
    CONSTRAINT CK_Customer_PhoneNumber CHECK (
        PhoneNumber IS NULL OR 
        PhoneNumber LIKE '([0-9][0-9][0-9]) [0-9][0-9][0-9]-[0-9][0-9][0-9][0-9]'
    ),
    -- US postal code: 5 digits or 5+4 format
    CONSTRAINT CK_Customer_PostalCode CHECK (
        PostalCode IS NULL OR
        PostalCode LIKE '[0-9][0-9][0-9][0-9][0-9]' OR
        PostalCode LIKE '[0-9][0-9][0-9][0-9][0-9]-[0-9][0-9][0-9][0-9]'
    )
);
```

## Conditional Constraints

```sql
CREATE TABLE dbo.Shipment (
    ShipmentId INT NOT NULL,
    OrderId INT NOT NULL,
    Status VARCHAR(20) NOT NULL,
    ShippedDate DATETIME2(0) NULL,
    DeliveredDate DATETIME2(0) NULL,
    TrackingNumber VARCHAR(50) NULL,
    
    CONSTRAINT PK_Shipment PRIMARY KEY (ShipmentId),
    -- If shipped, must have tracking number
    CONSTRAINT CK_Shipment_TrackingNumber CHECK (
        Status <> 'Shipped' OR TrackingNumber IS NOT NULL
    ),
    -- If delivered, must have both shipped and delivered dates
    CONSTRAINT CK_Shipment_DeliveredDate CHECK (
        Status <> 'Delivered' OR 
        (ShippedDate IS NOT NULL AND DeliveredDate IS NOT NULL)
    ),
    -- Delivered date must be after shipped date
    CONSTRAINT CK_Shipment_DateOrder CHECK (
        DeliveredDate IS NULL OR 
        ShippedDate IS NULL OR
        DeliveredDate >= ShippedDate
    )
);
```

## Computed Column Constraints

```sql
CREATE TABLE dbo.SalesOrder (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    Subtotal DECIMAL(18,2) NOT NULL,
    TaxPercent DECIMAL(5,2) NOT NULL,
    TaxAmount AS (Subtotal * TaxPercent / 100) PERSISTED,
    Total AS (Subtotal + (Subtotal * TaxPercent / 100)) PERSISTED,
    
    CONSTRAINT PK_SalesOrder PRIMARY KEY (OrderId),
    CONSTRAINT CK_SalesOrder_Subtotal CHECK (Subtotal >= 0),
    CONSTRAINT CK_SalesOrder_TaxPercent CHECK (TaxPercent >= 0 AND TaxPercent <= 100)
    -- Computed columns automatically maintain Total = Subtotal + TaxAmount
);
```

## Business Rule Constraints

```sql
CREATE TABLE dbo.Subscription (
    SubscriptionId INT NOT NULL,
    CustomerId INT NOT NULL,
    PlanType VARCHAR(20) NOT NULL,
    MonthlyPrice DECIMAL(18,2) NOT NULL,
    MaxUsers INT NOT NULL,
    MaxStorage INT NOT NULL,  -- GB
    
    CONSTRAINT PK_Subscription PRIMARY KEY (SubscriptionId),
    CONSTRAINT CK_Subscription_PlanType CHECK (
        PlanType IN ('Free', 'Basic', 'Professional', 'Enterprise')
    ),
    -- Free plan rules
    CONSTRAINT CK_Subscription_FreePlan CHECK (
        PlanType <> 'Free' OR 
        (MonthlyPrice = 0 AND MaxUsers = 1 AND MaxStorage = 5)
    ),
    -- Basic plan rules
    CONSTRAINT CK_Subscription_BasicPlan CHECK (
        PlanType <> 'Basic' OR
        (MonthlyPrice >= 9.99 AND MaxUsers <= 5 AND MaxStorage <= 50)
    ),
    -- Professional plan rules
    CONSTRAINT CK_Subscription_ProfessionalPlan CHECK (
        PlanType <> 'Professional' OR
        (MonthlyPrice >= 29.99 AND MaxUsers <= 20 AND MaxStorage <= 500)
    )
);
```

## Complex Business Logic

```sql
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    HireDate DATE NOT NULL,
    TerminationDate DATE NULL,
    IsActive BIT NOT NULL,
    Salary DECIMAL(18,2) NOT NULL,
    CommissionRate DECIMAL(5,2) NULL,
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId),
    -- Termination date must be after hire date
    CONSTRAINT CK_Employee_TerminationDate CHECK (
        TerminationDate IS NULL OR TerminationDate >= HireDate
    ),
    -- If terminated, cannot be active
    CONSTRAINT CK_Employee_ActiveStatus CHECK (
        TerminationDate IS NULL OR IsActive = 0
    ),
    -- Commission rate only valid when salary is above threshold
    CONSTRAINT CK_Employee_CommissionRate CHECK (
        CommissionRate IS NULL OR
        (Salary >= 50000 AND CommissionRate >= 0 AND CommissionRate <= 20)
    )
);
```

## Table-Level Constraints with Functions

```sql
-- Create function for complex validation
CREATE FUNCTION dbo.fn_IsValidCreditCard(@CardNumber VARCHAR(20))
RETURNS BIT
AS
BEGIN
    -- Simplified Luhn algorithm check
    DECLARE @IsValid BIT = 0;
    
    IF LEN(@CardNumber) = 16 AND @CardNumber NOT LIKE '%[^0-9]%'
    BEGIN
        SET @IsValid = 1;
    END;
    
    RETURN @IsValid;
END;
GO

CREATE TABLE dbo.Payment (
    PaymentId INT NOT NULL,
    OrderId INT NOT NULL,
    CardNumber VARCHAR(20) NOT NULL,
    Amount DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Payment PRIMARY KEY (PaymentId),
    CONSTRAINT CK_Payment_Amount CHECK (Amount > 0),
    CONSTRAINT CK_Payment_CardNumber CHECK (dbo.fn_IsValidCreditCard(CardNumber) = 1)
);
```

## Adding Constraints to Existing Tables

```sql
-- Add constraint with validation
ALTER TABLE dbo.Product
ADD CONSTRAINT CK_Product_Price CHECK (Price >= 0);

-- Add constraint without checking existing data (risky!)
ALTER TABLE dbo.Product WITH NOCHECK
ADD CONSTRAINT CK_Product_Price CHECK (Price >= 0);

-- Find existing violations before adding constraint
SELECT ProductId, Price
FROM dbo.Product
WHERE Price < 0;

-- Fix violations
UPDATE dbo.Product
SET Price = 0
WHERE Price < 0;

-- Now add constraint safely
ALTER TABLE dbo.Product
ADD CONSTRAINT CK_Product_Price CHECK (Price >= 0);
```

## Disabling and Enabling Constraints

```sql
-- Disable constraint for bulk load
ALTER TABLE dbo.Product NOCHECK CONSTRAINT CK_Product_Price;

-- Bulk insert (skips validation)
INSERT INTO dbo.Product (ProductId, Name, Price)
SELECT ProductId, Name, Price
FROM ExternalDataSource;

-- Re-enable and check existing data
ALTER TABLE dbo.Product CHECK CONSTRAINT CK_Product_Price;

-- Re-enable without checking (leaves violations)
ALTER TABLE dbo.Product WITH NOCHECK CHECK CONSTRAINT CK_Product_Price;
```

## Named vs Anonymous Constraints

```sql
-- ❌ Bad: Anonymous constraint (auto-generated name)
CREATE TABLE dbo.Product (
    ProductId INT PRIMARY KEY,
    Price DECIMAL(18,2) CHECK (Price >= 0)
);
-- Constraint name: something like "CK__Product__5EBF139D2F10007B"

-- ✅ Good: Named constraint (meaningful name)
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId),
    CONSTRAINT CK_Product_Price CHECK (Price >= 0)
);
-- Constraint name: "CK_Product_Price" (clear and predictable)
```

## Viewing Constraints

```sql
-- View all check constraints for a table
SELECT 
    cc.name AS ConstraintName,
    cc.definition AS ConstraintDefinition,
    cc.is_disabled,
    cc.is_not_trusted
FROM sys.check_constraints cc
INNER JOIN sys.tables t ON cc.parent_object_id = t.object_id
WHERE t.name = 'Product';

-- Find untrusted constraints (not validated)
SELECT 
    t.name AS TableName,
    cc.name AS ConstraintName,
    cc.definition
FROM sys.check_constraints cc
INNER JOIN sys.tables t ON cc.parent_object_id = t.object_id
WHERE cc.is_not_trusted = 1;
```

## Error Handling

```sql
CREATE PROCEDURE dbo.usp_CreateProduct
    @ProductId INT,
    @Name NVARCHAR(200),
    @Price DECIMAL(18,2),
    @StockQuantity INT
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        INSERT INTO dbo.Product (ProductId, Name, Price, StockQuantity)
        VALUES (@ProductId, @Name, @Price, @StockQuantity);
        
        SELECT 'SUCCESS' AS Result;
    END TRY
    BEGIN CATCH
        -- Check if constraint violation
        IF ERROR_NUMBER() = 547  -- CHECK constraint violation
        BEGIN
            DECLARE @ConstraintName NVARCHAR(128) = 
                CONSTRAINT_NAME FROM INFORMATION_SCHEMA.CONSTRAINT_COLUMN_USAGE
                WHERE TABLE_NAME = 'Product';
            
            RAISERROR('Invalid product data: %s', 16, 1, @ConstraintName);
        END
        ELSE
        BEGIN
            THROW;
        END;
    END CATCH;
END;
```

## Why Validation Only in Application Is a Problem

1. **Bypassable**: Database can be accessed directly via SQL tools
2. **Duplicated logic**: Every application must implement same rules
3. **Inconsistent**: Different applications may have different validation
4. **Runtime failures**: Bugs allow invalid data into database

## Symptoms

- Comments in code like "must be positive" or "cannot exceed 100"
- Data cleanup scripts to fix constraint violations
- Defensive code checking for impossible states
- Bug reports about invalid data in database

## Benefits

- **Last line of defense**: Rules enforced even if application fails
- **Self-documenting**: Schema reveals business rules
- **Consistent enforcement**: Single source of truth
- **Early failure**: Violations caught at insert/update time
- **Performance**: Validated at database level, no application roundtrip

## Trade-offs

- **Less flexibility**: Cannot temporarily store invalid data
- **Migration complexity**: Adding constraints to tables with bad data
- **Error messages**: Generic constraint violation messages (customize in application)
- **Function overhead**: Complex checks with functions slower than application logic

## When to Use

✅ Use check constraints for:
- Value ranges (age > 0, price >= 0)
- Enumerated values (status IN (...))
- Data format validation (phone, email patterns)
- Business invariants (end_date > start_date)
- Required field combinations

❌ Consider alternatives for:
- Complex cross-table validation (use triggers)
- Rules that change frequently (externalize to config)
- Calculations that need explanation (keep in application)

## See Also

- [Schema Constraints](./schema-constraints.md)
- [Foreign Key Constraints](./foreign-key-constraints.md)
- [Unique Constraints](./unique-constraints.md)
- [Named Constraints](./named-constraints.md)
- [Domain-Appropriate Types](./domain-appropriate-types.md)
