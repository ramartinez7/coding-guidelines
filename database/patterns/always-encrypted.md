# Always Encrypted

> Encrypt sensitive columns so data remains encrypted at rest and in transit—even DBAs cannot read plaintext values.

## Problem

Sensitive data like credit card numbers, social security numbers, and personal information is stored in plaintext. Database administrators, backups, and logs can expose this data. Traditional column-level encryption requires key management in the application.

## Example

### ❌ Before (Plaintext Sensitive Data)

```sql
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    SSN VARCHAR(11) NOT NULL,              -- ❌ Plaintext!
    CreditCardNumber VARCHAR(20) NOT NULL, -- ❌ Plaintext!
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

-- Data visible to anyone with SELECT permission
SELECT CustomerId, SSN, CreditCardNumber
FROM dbo.Customer;
-- Results: 123, 123-45-6789, 4532-1234-5678-9010
-- ❌ DBAs can see sensitive data in queries, backups, logs
```

### ✅ After (Always Encrypted)

```sql
-- Create Column Master Key (stored in certificate store, Azure Key Vault, etc.)
CREATE COLUMN MASTER KEY CMK_Auto1
WITH (
    KEY_STORE_PROVIDER_NAME = 'MSSQL_CERTIFICATE_STORE',
    KEY_PATH = 'CurrentUser/My/A1B2C3D4E5F6...'
);

-- Create Column Encryption Key (encrypted by CMK)
CREATE COLUMN ENCRYPTION KEY CEK_Auto1
WITH VALUES (
    COLUMN_MASTER_KEY = CMK_Auto1,
    ALGORITHM = 'RSA_OAEP',
    ENCRYPTED_VALUE = 0x016E000001630075007200...
);

-- Create table with encrypted columns
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastName NVARCHAR(100) NOT NULL,
    -- Encrypted columns
    SSN VARCHAR(11) 
        COLLATE Latin1_General_BIN2
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = CEK_Auto1,
            ENCRYPTION_TYPE = DETERMINISTIC,  -- Allows WHERE clause equality
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ) NOT NULL,
    CreditCardNumber VARCHAR(20)
        COLLATE Latin1_General_BIN2
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = CEK_Auto1,
            ENCRYPTION_TYPE = RANDOMIZED,  -- Most secure, no WHERE clause
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

-- Without access to keys, data appears encrypted
SELECT CustomerId, SSN, CreditCardNumber
FROM dbo.Customer;
-- Results: 123, 0x01AB02CD..., 0x03EF04GH...
-- ✅ DBAs see only encrypted values
```

## Encryption Types

### Deterministic Encryption

Same plaintext always produces same ciphertext—enables equality searches:

```sql
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    -- Deterministic: Can use in WHERE clause
    SSN VARCHAR(11)
        COLLATE Latin1_General_BIN2
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = CEK_Auto1,
            ENCRYPTION_TYPE = DETERMINISTIC,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ) NOT NULL,
    
    CONSTRAINT PK_Employee PRIMARY KEY (EmployeeId)
);

-- Application with correct keys can query by encrypted column
SELECT EmployeeId, FirstName
FROM dbo.Employee
WHERE SSN = '123-45-6789';  -- ✅ Works with deterministic encryption
```

### Randomized Encryption

Same plaintext produces different ciphertext—most secure but no searching:

```sql
CREATE TABLE dbo.Payment (
    PaymentId INT NOT NULL,
    CustomerId INT NOT NULL,
    -- Randomized: Cannot search, most secure
    CreditCardNumber VARCHAR(20)
        COLLATE Latin1_General_BIN2
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = CEK_Auto1,
            ENCRYPTION_TYPE = RANDOMIZED,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ) NOT NULL,
    Amount DECIMAL(18,2) NOT NULL,
    
    CONSTRAINT PK_Payment PRIMARY KEY (PaymentId)
);

-- Cannot query by randomized column
SELECT PaymentId
FROM dbo.Payment
WHERE CreditCardNumber = '4532-1234-5678-9010';
-- ❌ Error: Operand type clash
```

## Key Hierarchy

```sql
-- 1. Column Master Key (CMK)
-- Stored outside SQL Server: Certificate Store, Azure Key Vault, HSM
-- Encrypts Column Encryption Keys

CREATE COLUMN MASTER KEY CMK_AzureKeyVault
WITH (
    KEY_STORE_PROVIDER_NAME = 'AZURE_KEY_VAULT',
    KEY_PATH = 'https://myvault.vault.azure.net/keys/MyKey/abc123...'
);

-- 2. Column Encryption Key (CEK)  
-- Stored in SQL Server, encrypted by CMK
-- Encrypts actual column data

CREATE COLUMN ENCRYPTION KEY CEK_Auto1
WITH VALUES (
    COLUMN_MASTER_KEY = CMK_AzureKeyVault,
    ALGORITHM = 'RSA_OAEP',
    ENCRYPTED_VALUE = 0x016E000001630075007200...
);

-- 3. Encrypted Columns
-- Data encrypted with CEK

CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    SSN VARCHAR(11)
        COLLATE Latin1_General_BIN2
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = CEK_Auto1,
            ENCRYPTION_TYPE = DETERMINISTIC,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ),
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);
```

## Column Master Key Providers

### Windows Certificate Store

```sql
CREATE COLUMN MASTER KEY CMK_Certificate
WITH (
    KEY_STORE_PROVIDER_NAME = 'MSSQL_CERTIFICATE_STORE',
    KEY_PATH = 'CurrentUser/My/A1B2C3D4E5F6...'  -- Certificate thumbprint
);
```

### Azure Key Vault

```sql
CREATE COLUMN MASTER KEY CMK_AzureKeyVault
WITH (
    KEY_STORE_PROVIDER_NAME = 'AZURE_KEY_VAULT',
    KEY_PATH = 'https://myvault.vault.azure.net/keys/MyKey/version'
);
```

### Cryptographic Service Provider (CSP)

```sql
CREATE COLUMN MASTER KEY CMK_CSP
WITH (
    KEY_STORE_PROVIDER_NAME = 'MSSQL_CSP_PROVIDER',
    KEY_PATH = 'MyKeyContainer'
);
```

### Hardware Security Module (HSM)

```sql
CREATE COLUMN MASTER KEY CMK_CNG
WITH (
    KEY_STORE_PROVIDER_NAME = 'MSSQL_CNG_STORE',
    KEY_PATH = 'MyHSMProvider/MyKeyContainer'
);
```

## Adding Encryption to Existing Tables

```sql
-- Existing table
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    SSN VARCHAR(11) NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

-- Cannot directly encrypt existing column
-- Must create new encrypted column and migrate data

-- 1. Add new encrypted column
ALTER TABLE dbo.Customer
ADD SSN_Encrypted VARCHAR(11)
    COLLATE Latin1_General_BIN2
    ENCRYPTED WITH (
        COLUMN_ENCRYPTION_KEY = CEK_Auto1,
        ENCRYPTION_TYPE = DETERMINISTIC,
        ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
    ) NULL;

-- 2. Copy data (requires connection with encryption enabled)
UPDATE dbo.Customer
SET SSN_Encrypted = SSN;

-- 3. Drop old column
ALTER TABLE dbo.Customer DROP COLUMN SSN;

-- 4. Rename new column
EXEC sp_rename 'dbo.Customer.SSN_Encrypted', 'SSN', 'COLUMN';

-- 5. Make NOT NULL
ALTER TABLE dbo.Customer
ALTER COLUMN SSN VARCHAR(11)
    COLLATE Latin1_General_BIN2
    ENCRYPTED WITH (
        COLUMN_ENCRYPTION_KEY = CEK_Auto1,
        ENCRYPTION_TYPE = DETERMINISTIC,
        ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
    ) NOT NULL;
```

## Application Configuration

### Connection String

```csharp
// C# connection string
var connectionString = 
    "Server=myserver;Database=mydb;Column Encryption Setting=Enabled;";

using var connection = new SqlConnection(connectionString);
connection.Open();

// Application can now read/write encrypted columns transparently
var cmd = new SqlCommand(
    "SELECT CustomerId, SSN FROM dbo.Customer WHERE CustomerId = @Id", 
    connection
);
cmd.Parameters.AddWithValue("@Id", 123);

using var reader = cmd.ExecuteReader();
while (reader.Read())
{
    var ssn = reader.GetString(1);  // Decrypted automatically
    Console.WriteLine($"SSN: {ssn}");
}
```

### Column Encryption Parameter

```csharp
// Encrypt parameter value before sending to SQL
var cmd = new SqlCommand(
    "SELECT CustomerId FROM dbo.Customer WHERE SSN = @SSN",
    connection
);

var param = cmd.Parameters.AddWithValue("@SSN", "123-45-6789");
// Mark as encrypted for Always Encrypted
((SqlParameter)param).ForceColumnEncryption = true;

var customerId = (int)cmd.ExecuteScalar();
```

## Querying Encrypted Columns

### Supported Operations

```sql
-- Deterministic encryption supports:

-- Equality comparison
SELECT * FROM dbo.Customer
WHERE SSN = '123-45-6789';  -- ✅ Supported

-- IN clause
SELECT * FROM dbo.Customer
WHERE SSN IN ('123-45-6789', '987-65-4321');  -- ✅ Supported

-- JOIN on encrypted columns
SELECT c.CustomerId, o.OrderId
FROM dbo.Customer c
INNER JOIN dbo.Order o ON c.CustomerId = o.CustomerId
WHERE c.SSN = '123-45-6789';  -- ✅ Supported
```

### Unsupported Operations

```sql
-- Randomized encryption does not support any operations
-- Deterministic encryption does NOT support:

-- Range queries
SELECT * FROM dbo.Customer
WHERE SSN > '100-00-0000';  -- ❌ Not supported

-- LIKE pattern matching
SELECT * FROM dbo.Customer
WHERE SSN LIKE '123-%';  -- ❌ Not supported

-- Arithmetic operations
SELECT * FROM dbo.Payment
WHERE EncryptedAmount * 1.1 > 100;  -- ❌ Not supported

-- Aggregations
SELECT COUNT(DISTINCT SSN)
FROM dbo.Customer;  -- ❌ Not supported

-- ORDER BY
SELECT * FROM dbo.Customer
ORDER BY SSN;  -- ❌ Not supported
```

## Rich Queries Pattern

For complex queries, create indexed computed columns:

```sql
-- Cannot index encrypted column directly
-- Create hash for indexing and searching

CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    -- Encrypted column
    SSN VARCHAR(11)
        COLLATE Latin1_General_BIN2
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = CEK_Auto1,
            ENCRYPTION_TYPE = RANDOMIZED,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ) NOT NULL,
    -- Hash for searching (salted with customer-specific value)
    SSN_Hash AS HASHBYTES('SHA2_256', SSN + CAST(CustomerId AS VARCHAR)) PERSISTED,
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

-- Index the hash
CREATE NONCLUSTERED INDEX IX_Customer_SSN_Hash
ON dbo.Customer(SSN_Hash);

-- Search by hash
DECLARE @SearchSSN VARCHAR(11) = '123-45-6789';
DECLARE @SearchHash VARBINARY(32) = HASHBYTES('SHA2_256', @SearchSSN + CAST(@CustomerId AS VARCHAR));

SELECT CustomerId, FirstName
FROM dbo.Customer
WHERE SSN_Hash = @SearchHash;
```

## Viewing Encryption Metadata

```sql
-- View Column Master Keys
SELECT 
    name AS CMK_Name,
    key_store_provider_name,
    key_path
FROM sys.column_master_keys;

-- View Column Encryption Keys
SELECT 
    cek.name AS CEK_Name,
    cmk.name AS CMK_Name,
    cek.encryption_algorithm_name
FROM sys.column_encryption_keys cek
INNER JOIN sys.column_encryption_key_values cekv 
    ON cek.column_encryption_key_id = cekv.column_encryption_key_id
INNER JOIN sys.column_master_keys cmk 
    ON cekv.column_master_key_id = cmk.column_master_key_id;

-- View encrypted columns
SELECT 
    t.name AS TableName,
    c.name AS ColumnName,
    cek.name AS CEK_Name,
    c.encryption_type_desc,
    c.encryption_algorithm_name
FROM sys.columns c
INNER JOIN sys.tables t ON c.object_id = t.object_id
INNER JOIN sys.column_encryption_keys cek 
    ON c.column_encryption_key_id = cek.column_encryption_key_id
WHERE c.encryption_type IS NOT NULL
ORDER BY t.name, c.column_id;
```

## Key Rotation

```sql
-- Create new CEK
CREATE COLUMN ENCRYPTION KEY CEK_Auto2
WITH VALUES (
    COLUMN_MASTER_KEY = CMK_AzureKeyVault,
    ALGORITHM = 'RSA_OAEP',
    ENCRYPTED_VALUE = 0x...
);

-- Re-encrypt column with new key
ALTER TABLE dbo.Customer
ALTER COLUMN SSN VARCHAR(11)
    COLLATE Latin1_General_BIN2
    ENCRYPTED WITH (
        COLUMN_ENCRYPTION_KEY = CEK_Auto2,  -- New key
        ENCRYPTION_TYPE = DETERMINISTIC,
        ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
    );

-- Drop old key (after all columns re-encrypted)
DROP COLUMN ENCRYPTION KEY CEK_Auto1;
```

## Performance Considerations

```sql
-- Encryption adds overhead:
-- - 2-3x slower inserts/updates
-- - Encrypted columns cannot be indexed
-- - Increased column size (ciphertext larger than plaintext)

-- Optimize:
-- 1. Only encrypt sensitive columns
CREATE TABLE dbo.Customer (
    CustomerId INT NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,  -- Not encrypted (not sensitive)
    LastName NVARCHAR(100) NOT NULL,   -- Not encrypted
    SSN VARCHAR(11) ENCRYPTED WITH (...) NOT NULL,  -- Encrypted (sensitive)
    
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerId)
);

-- 2. Use deterministic for searchable columns
-- 3. Create hash columns for indexing
-- 4. Batch operations when possible
```

## Why Plaintext Sensitive Data Is a Problem

1. **Data breaches**: Backups, logs, and temp files expose data
2. **Insider threats**: DBAs have full access to sensitive data
3. **Compliance violations**: GDPR, PCI-DSS require encryption
4. **Audit logs**: Sensitive data appears in query logs
5. **Cloud concerns**: Data visible to cloud provider

## Symptoms

- Audit findings about unencrypted sensitive data
- Compliance requirements not met
- DBAs can see SSN, credit cards in queries
- Backup files contain plaintext sensitive data

## Benefits

- **End-to-end encryption**: Data encrypted at rest and in transit
- **Key isolation**: Keys outside database, even DBA cannot decrypt
- **Compliance**: Meets GDPR, PCI-DSS, HIPAA requirements
- **Transparent**: Application code largely unchanged
- **Performance**: Better than application-level encryption

## Trade-offs

- **Query limitations**: Cannot index, sort, or aggregate encrypted columns
- **Complexity**: Key management infrastructure required
- **Application changes**: Must enable Always Encrypted in connection
- **Performance**: 2-3x overhead on writes to encrypted columns
- **Collation**: Must use binary collation (BIN2)

## When to Use

✅ Use Always Encrypted for:
- SSN, credit card numbers, personal identifiers
- Medical records, financial data
- Compliance-driven encryption (PCI-DSS, HIPAA)
- Cloud scenarios where data must be encrypted

❌ Consider alternatives for:
- Data that needs complex queries (use Transparent Data Encryption)
- High-volume OLTP on encrypted columns
- When application-level encryption is sufficient

## See Also

- [Row-Level Security](./row-level-security.md)
- [SQL Injection Prevention](./sql-injection-prevention.md)
- [Least Privilege](./least-privilege.md)
- [Audit Columns](./audit-columns.md)
