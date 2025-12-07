# Schema Constraints

> Use database constraints to enforce business rules at the data layer—prevent invalid data from ever being persisted.

## Problem

When business rules are only enforced in application code, they can be bypassed by bugs, rogue queries, or multiple applications accessing the same database. Invalid data enters the system and corrupts the domain.

## Example

### ❌ Before

```sql
-- No constraints: relies on application code to validate
CREATE TABLE dbo.Order (
    OrderId INT PRIMARY KEY,
    CustomerId INT,
    Total DECIMAL(18,2),
    OrderDate DATETIME2,
    Status VARCHAR(20)
);

-- Application code must remember to check:
-- - CustomerId exists
-- - Total is non-negative
-- - OrderDate is provided
-- - Status is valid value
```

### ✅ After

```sql
-- Constraints enforce rules at the database level
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    OrderDate DATETIME2(0) NOT NULL DEFAULT SYSDATETIME(),
    Status VARCHAR(20) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerId) 
        REFERENCES dbo.Customer(CustomerId),
    CONSTRAINT CK_Order_Total CHECK (Total >= 0),
    CONSTRAINT CK_Order_Status CHECK (Status IN ('Pending', 'Processing', 'Shipped', 'Delivered', 'Cancelled')),
    CONSTRAINT DF_Order_Status DEFAULT 'Pending' FOR Status
);

-- Now invalid data is impossible:
-- ❌ Cannot insert order with negative total
INSERT INTO dbo.Order (OrderId, CustomerId, Total, Status)
VALUES (1, 101, -50.00, 'Pending');
-- Error: The INSERT statement conflicted with the CHECK constraint "CK_Order_Total"

-- ❌ Cannot insert order with invalid status
INSERT INTO dbo.Order (OrderId, CustomerId, Total, Status)
VALUES (2, 101, 50.00, 'InvalidStatus');
-- Error: The INSERT statement conflicted with the CHECK constraint "CK_Order_Status"

-- ❌ Cannot insert order with non-existent customer
INSERT INTO dbo.Order (OrderId, CustomerId, Total, Status)
VALUES (3, 999, 50.00, 'Pending');
-- Error: The INSERT statement conflicted with the FOREIGN KEY constraint "FK_Order_Customer"
```

## Constraint Types

### NOT NULL Constraint

Prevent null values in columns where a value must always exist.

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,        -- Email always required
    Phone NVARCHAR(20) NULL,              -- Phone is optional
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);
```

### CHECK Constraint

Enforce value ranges or specific conditions.

```sql
CREATE TABLE dbo.Product (
    ProductId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    StockQuantity INT NOT NULL,
    DiscountPercent DECIMAL(5,2) NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (ProductId),
    CONSTRAINT CK_Product_Price CHECK (Price >= 0),
    CONSTRAINT CK_Product_StockQuantity CHECK (StockQuantity >= 0),
    CONSTRAINT CK_Product_DiscountPercent CHECK (DiscountPercent >= 0 AND DiscountPercent <= 100)
);
```

### UNIQUE Constraint

Ensure uniqueness of values beyond the primary key.

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    Username NVARCHAR(50) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId),
    CONSTRAINT UQ_Customer_Email UNIQUE (Email),
    CONSTRAINT UQ_Customer_Username UNIQUE (Username)
);

-- Now duplicate emails are impossible
INSERT INTO dbo.Customer (CustomerId, Email, Username)
VALUES (1, 'alice@example.com', 'alice');  -- OK

INSERT INTO dbo.Customer (CustomerId, Email, Username)
VALUES (2, 'alice@example.com', 'alice2');  -- ❌ Error: duplicate email
```

### FOREIGN KEY Constraint

Enforce referential integrity between tables.

```sql
CREATE TABLE dbo.OrderLineItem (
    OrderLineItemId INT NOT NULL,
    OrderId INT NOT NULL,
    ProductId INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_OrderLineItem PRIMARY KEY (OrderLineItemId),
    CONSTRAINT FK_OrderLineItem_Order FOREIGN KEY (OrderId)
        REFERENCES dbo.Order(OrderId) ON DELETE CASCADE,
    CONSTRAINT FK_OrderLineItem_Product FOREIGN KEY (ProductId)
        REFERENCES dbo.Product(ProductId),
    CONSTRAINT CK_OrderLineItem_Quantity CHECK (Quantity > 0),
    CONSTRAINT CK_OrderLineItem_UnitPrice CHECK (UnitPrice >= 0)
);
```

### DEFAULT Constraint

Provide default values when none is specified.

```sql
CREATE TABLE dbo.Order (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    Total DECIMAL(18,2) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    CreatedAt DATETIME2(0) NOT NULL,
    
    CONSTRAINT PK_Order PRIMARY KEY (OrderId),
    CONSTRAINT DF_Order_Status DEFAULT 'Pending' FOR Status,
    CONSTRAINT DF_Order_CreatedAt DEFAULT SYSDATETIME() FOR CreatedAt
);

-- CreatedAt and Status automatically filled
INSERT INTO dbo.Order (OrderId, CustomerId, Total)
VALUES (1, 101, 99.99);
-- Status = 'Pending', CreatedAt = current timestamp
```

## Why It's a Problem

1. **Fragile validation**: Application code can be bypassed by bugs, admin tools, or direct database access
2. **Scattered rules**: Same validation logic duplicated across multiple applications or services
3. **Runtime failures**: Invalid data discovered after being persisted, requiring data cleanup
4. **Data corruption**: Inconsistent state makes queries and reports unreliable

## Symptoms

- Comments in code saying "must be positive" or "required field"
- Data cleanup scripts to fix constraint violations
- Application crashes due to unexpected null or invalid values
- Multiple applications sharing a database with inconsistent validation

## Benefits

- **Last line of defense**: Rules enforced even if application code fails
- **Self-documenting schema**: Constraints communicate business rules to anyone reading the schema
- **Better error messages**: Constraint violations fail fast at insertion time with clear messages
- **Data integrity**: Impossible to have invalid state in the database

## Trade-offs

- **Performance overhead**: Each constraint adds validation work during writes (minimal in practice)
- **Less flexibility**: Cannot temporarily store invalid data during complex operations (generally a good thing)
- **Migration complexity**: Adding constraints to existing tables with invalid data requires cleanup first

## See Also

- [Check Constraints](./check-constraints.md)
- [Foreign Key Constraints](./foreign-key-constraints.md)
- [Unique Constraints](./unique-constraints.md)
- [Named Constraints](./named-constraints.md)
