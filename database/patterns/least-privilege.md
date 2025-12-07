# Least Privilege

> Grant only the minimum permissions needed—reduce blast radius of compromised accounts and prevent unauthorized access.

## Problem

Database users have excessive permissions like `db_owner` or `db_datareader` when they only need access to specific tables and operations. Compromised accounts can access or modify data they shouldn't touch.

## Example

### ❌ Before (Excessive Permissions)

```sql
-- Application account with full database access
CREATE LOGIN AppUser WITH PASSWORD = 'P@ssw0rd!';
GO

USE MyDatabase;
GO

CREATE USER AppUser FOR LOGIN AppUser;
GO

-- ❌ Grants ALL permissions on database
EXEC sp_addrolemember 'db_owner', 'AppUser';

-- AppUser can now:
-- - Read ALL tables
-- - Modify ALL tables  
-- - Drop tables
-- - Create new tables
-- - Change schema
-- - View other users' data
```

### ✅ After (Least Privilege)

```sql
-- Application account with minimal permissions
CREATE LOGIN AppUser WITH PASSWORD = 'P@ssw0rd!';
GO

USE MyDatabase;
GO

CREATE USER AppUser FOR LOGIN AppUser;
GO

-- Grant only specific permissions needed
GRANT SELECT ON dbo.Customer TO AppUser;
GRANT SELECT ON dbo.Order TO AppUser;
GRANT INSERT ON dbo.Order TO AppUser;
GRANT UPDATE ON dbo.Order TO AppUser;
GRANT EXECUTE ON dbo.usp_CreateOrder TO AppUser;
GRANT EXECUTE ON dbo.usp_UpdateOrderStatus TO AppUser;

-- AppUser can now:
-- ✅ Read Customer and Order tables
-- ✅ Insert/Update orders
-- ✅ Execute specific procedures
-- ❌ Cannot access other tables
-- ❌ Cannot drop or alter tables
-- ❌ Cannot create new objects
```

## Principle of Least Privilege

### Read-Only Application

```sql
-- Read-only reporting account
CREATE LOGIN ReportUser WITH PASSWORD = 'P@ssw0rd!';
GO

USE MyDatabase;
GO

CREATE USER ReportUser FOR LOGIN ReportUser;
GO

-- Grant SELECT only on specific tables
GRANT SELECT ON dbo.Customer TO ReportUser;
GRANT SELECT ON dbo.Order TO ReportUser;
GRANT SELECT ON dbo.OrderLineItem TO ReportUser;
GRANT SELECT ON dbo.Product TO ReportUser;

-- Or use view to restrict columns
CREATE VIEW dbo.vw_CustomerReport AS
SELECT 
    CustomerId,
    FirstName,
    LastName,
    City,
    State
    -- Exclude sensitive columns like SSN, CreditCard
FROM dbo.Customer;
GO

GRANT SELECT ON dbo.vw_CustomerReport TO ReportUser;
```

### Write-Only Application

```sql
-- Audit log writer (can only insert)
CREATE LOGIN AuditWriter WITH PASSWORD = 'P@ssw0rd!';
GO

USE MyDatabase;
GO

CREATE USER AuditWriter FOR LOGIN AuditWriter;
GO

-- Only INSERT permission
GRANT INSERT ON dbo.AuditLog TO AuditWriter;

-- Cannot read or modify existing audit logs
-- DENY SELECT ON dbo.AuditLog TO AuditWriter;  -- Explicit deny
-- DENY UPDATE ON dbo.AuditLog TO AuditWriter;
-- DENY DELETE ON dbo.AuditLog TO AuditWriter;
```

## Database Roles

### Built-in Roles

```sql
-- Fixed database roles (avoid when possible)
-- db_owner          - Full control (❌ Too much!)
-- db_datareader     - SELECT on all tables (often too much)
-- db_datawriter     - INSERT/UPDATE/DELETE on all tables (often too much)
-- db_ddladmin       - Can modify schema (usually not needed)
-- db_securityadmin  - Can manage permissions
-- db_accessadmin    - Can add/remove users
-- db_backupoperator - Can backup database
-- db_denydatareader - Explicit deny SELECT
-- db_denydatawriter - Explicit deny writes

-- Example: Backup operator
CREATE USER BackupUser FOR LOGIN BackupUser;
EXEC sp_addrolemember 'db_backupoperator', 'BackupUser';
```

### Custom Roles

```sql
-- Create custom roles for specific use cases
CREATE ROLE OrderManager;
GO

-- Grant permissions to role
GRANT SELECT ON dbo.Customer TO OrderManager;
GRANT SELECT ON dbo.Product TO OrderManager;
GRANT SELECT, INSERT, UPDATE ON dbo.Order TO OrderManager;
GRANT SELECT, INSERT, UPDATE ON dbo.OrderLineItem TO OrderManager;
GRANT EXECUTE ON dbo.usp_CreateOrder TO OrderManager;
GRANT EXECUTE ON dbo.usp_UpdateOrderStatus TO OrderManager;
GO

-- Add users to role
CREATE USER OrderAppUser FOR LOGIN OrderAppUser;
EXEC sp_addrolemember 'OrderManager', 'OrderAppUser';
GO

-- Now OrderAppUser inherits all OrderManager permissions
```

## Stored Procedure Security

### Ownership Chaining

```sql
-- User doesn't need permissions on tables if procedure owned by same schema
CREATE PROCEDURE dbo.usp_GetCustomerOrders
    @CustomerId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- User needs EXECUTE on procedure, not SELECT on tables
    SELECT 
        o.OrderId,
        o.OrderDate,
        o.Total
    FROM dbo.Order o
    WHERE o.CustomerId = @CustomerId;
END;
GO

-- Grant EXECUTE only
GRANT EXECUTE ON dbo.usp_GetCustomerOrders TO AppUser;

-- AppUser can call procedure but cannot directly query Order table
```

### EXECUTE AS

```sql
-- Procedure runs with elevated permissions
CREATE PROCEDURE dbo.usp_AdminOperation
WITH EXECUTE AS 'dbo'  -- Run as database owner
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Perform admin operation
    TRUNCATE TABLE dbo.TempTable;
END;
GO

-- Grant EXECUTE to regular user
GRANT EXECUTE ON dbo.usp_AdminOperation TO AppUser;

-- AppUser can execute procedure (which runs as dbo)
-- but cannot directly TRUNCATE tables
```

## Application Roles

```sql
-- Create application role (password-based)
CREATE APPLICATION ROLE OrderProcessingRole
WITH PASSWORD = 'SecurePassword123!';
GO

-- Grant permissions to app role
GRANT SELECT ON dbo.Customer TO OrderProcessingRole;
GRANT SELECT, INSERT, UPDATE ON dbo.Order TO OrderProcessingRole;
GO

-- Application activates role
-- SQL:
-- EXEC sp_setapprole 'OrderProcessingRole', 'SecurePassword123!';

-- C#:
using (var conn = new SqlConnection(connectionString))
{
    conn.Open();
    
    var cmd = new SqlCommand("sp_setapprole", conn);
    cmd.CommandType = CommandType.StoredProcedure;
    cmd.Parameters.AddWithValue("@rolename", "OrderProcessingRole");
    cmd.Parameters.AddWithValue("@password", "SecurePassword123!");
    cmd.ExecuteNonQuery();
    
    // Now have OrderProcessingRole permissions
}
```

## Schema-Level Permissions

```sql
-- Grant permissions on entire schema
CREATE SCHEMA Sales;
GO

CREATE TABLE Sales.Order (...);
CREATE TABLE Sales.OrderLineItem (...);
GO

-- Grant SELECT on all objects in schema
GRANT SELECT ON SCHEMA::Sales TO AppUser;

-- Grant EXECUTE on all procedures in schema
GRANT EXECUTE ON SCHEMA::Sales TO AppUser;
```

## Column-Level Permissions

```sql
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    Salary DECIMAL(18,2) NOT NULL,  -- Sensitive
    SSN VARCHAR(11) NOT NULL,        -- Sensitive
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId)
);
GO

-- Grant SELECT on specific columns only
GRANT SELECT (EmployeeId, FirstName, LastName, Email) 
ON dbo.Employee TO AppUser;

-- AppUser can read basic info but not Salary or SSN
SELECT FirstName, LastName, Email
FROM dbo.Employee;  -- ✅ Allowed

SELECT Salary
FROM dbo.Employee;  -- ❌ Permission denied
```

## Service Accounts

```sql
-- Different accounts for different services
CREATE LOGIN WebAppUser WITH PASSWORD = 'P@ssw0rd!';
CREATE LOGIN BatchProcessUser WITH PASSWORD = 'P@ssw0rd!';
CREATE LOGIN ReportingUser WITH PASSWORD = 'P@ssw0rd!';
GO

USE MyDatabase;
GO

-- Web app: Read and write orders
CREATE USER WebAppUser FOR LOGIN WebAppUser;
GRANT SELECT ON dbo.Customer TO WebAppUser;
GRANT SELECT, INSERT, UPDATE ON dbo.Order TO WebAppUser;
GO

-- Batch process: Update inventory
CREATE USER BatchProcessUser FOR LOGIN BatchProcessUser;
GRANT SELECT, UPDATE ON dbo.Product TO BatchProcessUser;
GRANT EXECUTE ON dbo.usp_UpdateInventory TO BatchProcessUser;
GO

-- Reporting: Read-only
CREATE USER ReportingUser FOR LOGIN ReportingUser;
GRANT SELECT ON SCHEMA::dbo TO ReportingUser;
GO
```

## Auditing Permissions

```sql
-- View database permissions for a user
SELECT 
    pr.name AS UserName,
    pr.type_desc AS UserType,
    pe.permission_name,
    pe.state_desc,
    OBJECT_SCHEMA_NAME(pe.major_id) + '.' + OBJECT_NAME(pe.major_id) AS ObjectName
FROM sys.database_principals pr
INNER JOIN sys.database_permissions pe ON pr.principal_id = pe.grantee_principal_id
WHERE pr.name = 'AppUser'
ORDER BY pe.permission_name, ObjectName;

-- View role memberships
SELECT 
    USER_NAME(rm.member_principal_id) AS UserName,
    USER_NAME(rm.role_principal_id) AS RoleName
FROM sys.database_role_members rm
WHERE USER_NAME(rm.member_principal_id) = 'AppUser';

-- Find users with excessive permissions
SELECT 
    pr.name AS UserName,
    pr.type_desc,
    COUNT(*) AS PermissionCount
FROM sys.database_principals pr
LEFT JOIN sys.database_permissions pe ON pr.principal_id = pe.grantee_principal_id
WHERE pr.type IN ('S', 'U')  -- SQL users and Windows users
  AND pr.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
GROUP BY pr.name, pr.type_desc
ORDER BY PermissionCount DESC;
```

## Row-Level Security Integration

```sql
-- Combine least privilege with RLS
CREATE FUNCTION dbo.fn_SecurityPredicate(@TenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS Result
WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT);
GO

CREATE SECURITY POLICY TenantFilter
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(TenantId) ON dbo.Order
WITH (STATE = ON);
GO

-- User still needs SELECT permission
GRANT SELECT ON dbo.Order TO AppUser;

-- But RLS ensures they only see their tenant's data
EXEC sp_set_session_context @key = N'TenantId', @value = 123;
SELECT * FROM dbo.Order;  -- Only sees TenantId = 123
```

## Temporal Table Permissions

```sql
-- Control access to historical data separately
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    Name NVARCHAR(200) NOT NULL,
    Email NVARCHAR(255) NOT NULL,
    ValidFrom DATETIME2(0) GENERATED ALWAYS AS ROW START NOT NULL,
    ValidTo DATETIME2(0) GENERATED ALWAYS AS ROW END NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo),
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.CustomerHistory));
GO

-- Grant access to current table only
GRANT SELECT ON dbo.Customer TO AppUser;

-- History table requires separate permission
-- GRANT SELECT ON dbo.CustomerHistory TO HistoricalReportUser;
```

## Dynamic Data Masking

```sql
-- Combine least privilege with data masking
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    Email NVARCHAR(255) MASKED WITH (FUNCTION = 'email()') NOT NULL,
    Phone VARCHAR(20) MASKED WITH (FUNCTION = 'partial(0,"XXX-XXX-",4)') NULL,
    SSN VARCHAR(11) MASKED WITH (FUNCTION = 'default()') NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);
GO

-- Regular user sees masked data
GRANT SELECT ON dbo.Customer TO AppUser;

SELECT * FROM dbo.Customer;
-- Email: aXXX@XXXX.com
-- Phone: XXX-XXX-1234
-- SSN: XXXX

-- Admin sees unmasked data
GRANT UNMASK TO AdminUser;
```

## Connection Security

```sql
-- Restrict connections by IP
-- (Configured at server/firewall level, not T-SQL)

-- Enforce SSL/TLS
-- ALTER LOGIN AppUser REQUIRE SSL;  -- Server 2016+

-- Restrict to specific client applications
-- (Use connection strings, not enforced by SQL)

-- Use Azure AD authentication instead of SQL authentication
-- CREATE USER [user@domain.com] FROM EXTERNAL PROVIDER;
```

## Deployment Accounts vs Runtime Accounts

```sql
-- Deployment account: Can modify schema
CREATE LOGIN DeploymentUser WITH PASSWORD = 'P@ssw0rd!';
GO

USE MyDatabase;
GO

CREATE USER DeploymentUser FOR LOGIN DeploymentUser;
EXEC sp_addrolemember 'db_ddladmin', 'DeploymentUser';
-- Can: CREATE/ALTER/DROP tables, procedures
-- Cannot: Read/write data (unless also granted)

-- Runtime account: Can access data only
CREATE LOGIN AppUser WITH PASSWORD = 'P@ssw0rd!';
GO

CREATE USER AppUser FOR LOGIN AppUser;
GRANT SELECT, INSERT, UPDATE ON dbo.Order TO AppUser;
-- Can: Read/write specific tables
-- Cannot: Modify schema
```

## Why Excessive Permissions Are a Problem

1. **Security breaches**: Compromised account accesses all data
2. **Data leaks**: Application bug exposes unintended data
3. **Accidental damage**: Developer drops production table
4. **Compliance violations**: Unnecessary access to sensitive data
5. **Audit failures**: Cannot prove least privilege

## Symptoms

- All application accounts have `db_owner`
- Single account for all applications
- Developers use production accounts
- No role-based access control
- "It's easier to give everyone full access"

## Benefits

- **Reduced blast radius**: Compromised account has limited access
- **Compliance**: Meets SOC2, PCI-DSS requirements
- **Auditability**: Clear what each account can do
- **Defense in depth**: Multiple layers of security
- **Easier debugging**: Permission errors reveal access attempts

## Trade-offs

- **Initial setup**: More work to configure granular permissions
- **Maintenance**: Must update permissions when adding features
- **Complexity**: More accounts and roles to manage
- **Debugging**: Permission errors during development

## When to Use

✅ Always use least privilege for:
- Production environments
- Multi-tenant applications
- Applications handling sensitive data
- Compliance-driven scenarios

❌ Might relax for:
- Local development (still be cautious)
- Isolated test environments
- Single-user applications

## Best Practices

1. **One account per service**: Don't share credentials
2. **Use roles**: Group permissions, assign users to roles
3. **Deny by default**: Grant only what's needed
4. **Regular audits**: Review and remove unused permissions
5. **Separate admin accounts**: Different accounts for admin tasks
6. **Document permissions**: Explain why each permission is granted
7. **Use stored procedures**: Limit direct table access
8. **Rotate credentials**: Change passwords regularly

## See Also

- [Row-Level Security](./row-level-security.md)
- [Always Encrypted](./always-encrypted.md)
- [SQL Injection Prevention](./sql-injection-prevention.md)
- [Audit Columns](./audit-columns.md)
