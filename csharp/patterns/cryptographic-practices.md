# Cryptographic Best Practices (Type-Safe Cryptography)

> Using weak algorithms, hardcoded keys, or manual crypto implementation—use modern cryptographic types and key management to prevent security vulnerabilities.

## Problem

Cryptographic code is notoriously difficult to implement correctly. Common mistakes include using weak algorithms (MD5, SHA1), hardcoded keys, improper IV generation, ECB mode, and manual implementation of crypto primitives. These mistakes lead to data breaches and compromised systems.

## Example

### ❌ Before

```csharp
public class LegacyCryptoService
{
    // Hardcoded key—compromised if source code leaks
    private const string HardcodedKey = "MySecretKey12345";
    
    public string Encrypt(string plaintext)
    {
        // Using weak algorithm (DES is deprecated)
        using var des = DES.Create();
        des.Key = Encoding.UTF8.GetBytes(HardcodedKey);
        des.Mode = CipherMode.ECB;  // ECB mode is insecure
        
        using var encryptor = des.CreateEncryptor();
        var bytes = Encoding.UTF8.GetBytes(plaintext);
        var encrypted = encryptor.TransformFinalBlock(bytes, 0, bytes.Length);
        
        return Convert.ToBase64String(encrypted);
    }
    
    public string Hash(string password)
    {
        // Using weak hash (MD5 is broken)
        using var md5 = MD5.Create();
        var bytes = Encoding.UTF8.GetBytes(password);
        var hash = md5.ComputeHash(bytes);
        
        return Convert.ToBase64String(hash);
    }
    
    public bool VerifyPassword(string password, string hash)
    {
        // Timing attack vulnerability
        var computedHash = Hash(password);
        return computedHash == hash;  // String comparison leaks timing info
    }
    
    public string GenerateToken()
    {
        // Using non-cryptographic RNG
        var random = new Random();
        var bytes = new byte[16];
        random.NextBytes(bytes);
        
        return Convert.ToBase64String(bytes);
    }
}
```

**Problems:**
- Hardcoded encryption keys
- Weak algorithms (DES, MD5, SHA1)
- Insecure cipher modes (ECB)
- No salt for password hashing
- Timing attacks in comparison
- Non-cryptographic RNG for security tokens

### ✅ After

```csharp
/// <summary>
/// Encrypted data with associated authentication tag.
/// Cannot be constructed without encryption.
/// </summary>
public sealed record EncryptedData
{
    public byte[] Ciphertext { get; }
    public byte[] Nonce { get; }
    public byte[] Tag { get; }
    
    internal EncryptedData(byte[] ciphertext, byte[] nonce, byte[] tag)
    {
        Ciphertext = ciphertext;
        Nonce = nonce;
        Tag = tag;
    }
}

/// <summary>
/// Cryptographic key with secure storage.
/// Cannot be constructed directly—must be generated or loaded securely.
/// </summary>
public sealed class EncryptionKey : IDisposable
{
    private readonly byte[] _key;
    private bool _disposed;
    
    private EncryptionKey(byte[] key)
    {
        _key = key;
    }
    
    /// <summary>
    /// Generates a new cryptographically secure key.
    /// </summary>
    public static EncryptionKey Generate()
    {
        // AES-256 requires 32 bytes
        var key = RandomNumberGenerator.GetBytes(32);
        return new EncryptionKey(key);
    }
    
    /// <summary>
    /// Loads a key from secure storage (e.g., Azure Key Vault, AWS KMS).
    /// </summary>
    public static async Task<Result<EncryptionKey, string>> LoadFromVaultAsync(
        string keyName,
        IKeyVaultClient keyVault)
    {
        try
        {
            var keyBytes = await keyVault.GetKeyAsync(keyName);
            return Result<EncryptionKey, string>.Success(new EncryptionKey(keyBytes));
        }
        catch (Exception ex)
        {
            return Result<EncryptionKey, string>.Failure(
                $"Failed to load key: {ex.Message}");
        }
    }
    
    /// <summary>
    /// Exports key for secure storage. Use only for key backup/migration.
    /// </summary>
    internal byte[] Export()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        return _key.ToArray();  // Return a copy
    }
    
    internal byte[] GetKey()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        return _key;
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            // Zero out key material
            Array.Clear(_key, 0, _key.Length);
            _disposed = true;
        }
    }
}

/// <summary>
/// Secure encryption service using modern algorithms.
/// </summary>
public sealed class SecureCryptoService
{
    private readonly EncryptionKey _key;
    
    public SecureCryptoService(EncryptionKey key)
    {
        _key = key;
    }
    
    /// <summary>
    /// Encrypts data using AES-256-GCM (authenticated encryption).
    /// </summary>
    public Result<EncryptedData, string> Encrypt(byte[] plaintext)
    {
        try
        {
            // Generate random nonce (12 bytes for GCM)
            var nonce = RandomNumberGenerator.GetBytes(12);
            
            // AES-GCM provides authenticated encryption
            using var aes = new AesGcm(_key.GetKey());
            
            var ciphertext = new byte[plaintext.Length];
            var tag = new byte[16];  // 128-bit authentication tag
            
            aes.Encrypt(nonce, plaintext, ciphertext, tag);
            
            return Result<EncryptedData, string>.Success(
                new EncryptedData(ciphertext, nonce, tag));
        }
        catch (Exception ex)
        {
            return Result<EncryptedData, string>.Failure(
                $"Encryption failed: {ex.Message}");
        }
    }
    
    /// <summary>
    /// Decrypts and authenticates data.
    /// </summary>
    public Result<byte[], string> Decrypt(EncryptedData encrypted)
    {
        try
        {
            using var aes = new AesGcm(_key.GetKey());
            
            var plaintext = new byte[encrypted.Ciphertext.Length];
            
            aes.Decrypt(
                encrypted.Nonce,
                encrypted.Ciphertext,
                encrypted.Tag,
                plaintext);
            
            return Result<byte[], string>.Success(plaintext);
        }
        catch (CryptographicException)
        {
            return Result<byte[], string>.Failure(
                "Decryption failed: data may be corrupted or tampered");
        }
        catch (Exception ex)
        {
            return Result<byte[], string>.Failure(
                $"Decryption error: {ex.Message}");
        }
    }
}

/// <summary>
/// Password hash with secure salt and algorithm.
/// </summary>
public sealed record PasswordHash
{
    public string Hash { get; }
    public int Iterations { get; }
    
    private PasswordHash(string hash, int iterations)
    {
        Hash = hash;
        Iterations = iterations;
    }
    
    /// <summary>
    /// Hashes a password using PBKDF2 with random salt.
    /// </summary>
    public static PasswordHash Create(string password, int iterations = 100_000)
    {
        if (iterations < 10_000)
            throw new ArgumentException("Iterations must be at least 10,000");
        
        // Generate random salt
        var salt = RandomNumberGenerator.GetBytes(16);
        
        // Use PBKDF2 (recommended for passwords)
        using var pbkdf2 = new Rfc2898DeriveBytes(
            password,
            salt,
            iterations,
            HashAlgorithmName.SHA256);
        
        var hash = pbkdf2.GetBytes(32);
        
        // Combine salt + hash for storage
        var combined = new byte[salt.Length + hash.Length];
        Buffer.BlockCopy(salt, 0, combined, 0, salt.Length);
        Buffer.BlockCopy(hash, 0, combined, salt.Length, hash.Length);
        
        return new PasswordHash(
            Convert.ToBase64String(combined),
            iterations);
    }
    
    /// <summary>
    /// Verifies password using constant-time comparison.
    /// </summary>
    public bool Verify(string password)
    {
        try
        {
            var combined = Convert.FromBase64String(Hash);
            
            // Extract salt and hash
            var salt = new byte[16];
            var storedHash = new byte[32];
            
            Buffer.BlockCopy(combined, 0, salt, 0, salt.Length);
            Buffer.BlockCopy(combined, salt.Length, storedHash, 0, storedHash.Length);
            
            // Hash the provided password with the same salt
            using var pbkdf2 = new Rfc2898DeriveBytes(
                password,
                salt,
                Iterations,
                HashAlgorithmName.SHA256);
            
            var computedHash = pbkdf2.GetBytes(32);
            
            // Constant-time comparison to prevent timing attacks
            return CryptographicOperations.FixedTimeEquals(computedHash, storedHash);
        }
        catch
        {
            return false;
        }
    }
}

/// <summary>
/// Cryptographically secure token generator.
/// </summary>
public static class SecureTokenGenerator
{
    /// <summary>
    /// Generates a cryptographically secure random token.
    /// </summary>
    public static string Generate(int byteCount = 32)
    {
        if (byteCount < 16)
            throw new ArgumentException("Token must be at least 16 bytes");
        
        var bytes = RandomNumberGenerator.GetBytes(byteCount);
        return Convert.ToBase64String(bytes);
    }
    
    /// <summary>
    /// Generates a URL-safe random token.
    /// </summary>
    public static string GenerateUrlSafe(int byteCount = 32)
    {
        var token = Generate(byteCount);
        
        // Make URL-safe by replacing characters
        return token
            .Replace("+", "-")
            .Replace("/", "_")
            .TrimEnd('=');
    }
}
```

## Advanced Patterns

### Envelope Encryption (Data Encryption Keys)

```csharp
/// <summary>
/// Envelope encryption: encrypt data with DEK, encrypt DEK with KEK.
/// </summary>
public sealed class EnvelopeEncryption
{
    private readonly EncryptionKey _keyEncryptionKey;
    
    public EnvelopeEncryption(EncryptionKey kek)
    {
        _keyEncryptionKey = kek;
    }
    
    public sealed record EncryptedEnvelope
    {
        public required EncryptedData EncryptedData { get; init; }
        public required EncryptedData EncryptedDataKey { get; init; }
    }
    
    /// <summary>
    /// Encrypts data using envelope encryption.
    /// Generates a random DEK, encrypts data with DEK, encrypts DEK with KEK.
    /// </summary>
    public Result<EncryptedEnvelope, string> Encrypt(byte[] plaintext)
    {
        try
        {
            // Generate random data encryption key
            using var dataKey = EncryptionKey.Generate();
            var dataKeyBytes = dataKey.Export();
            
            // Encrypt data with DEK
            var dataCrypto = new SecureCryptoService(dataKey);
            var encryptedDataResult = dataCrypto.Encrypt(plaintext);
            
            if (!encryptedDataResult.IsSuccess)
                return Result<EncryptedEnvelope, string>.Failure(
                    encryptedDataResult.Error!);
            
            // Encrypt DEK with KEK
            var keyCrypto = new SecureCryptoService(_keyEncryptionKey);
            var encryptedKeyResult = keyCrypto.Encrypt(dataKeyBytes);
            
            if (!encryptedKeyResult.IsSuccess)
                return Result<EncryptedEnvelope, string>.Failure(
                    encryptedKeyResult.Error!);
            
            // Zero out data key
            Array.Clear(dataKeyBytes, 0, dataKeyBytes.Length);
            
            return Result<EncryptedEnvelope, string>.Success(
                new EncryptedEnvelope
                {
                    EncryptedData = encryptedDataResult.Value!,
                    EncryptedDataKey = encryptedKeyResult.Value!
                });
        }
        catch (Exception ex)
        {
            return Result<EncryptedEnvelope, string>.Failure(
                $"Envelope encryption failed: {ex.Message}");
        }
    }
    
    /// <summary>
    /// Decrypts envelope: decrypt DEK with KEK, decrypt data with DEK.
    /// </summary>
    public Result<byte[], string> Decrypt(EncryptedEnvelope envelope)
    {
        try
        {
            // Decrypt DEK with KEK
            var keyCrypto = new SecureCryptoService(_keyEncryptionKey);
            var dataKeyResult = keyCrypto.Decrypt(envelope.EncryptedDataKey);
            
            if (!dataKeyResult.IsSuccess)
                return Result<byte[], string>.Failure(dataKeyResult.Error!);
            
            var dataKeyBytes = dataKeyResult.Value!;
            
            // Decrypt data with DEK
            using var dataKey = EncryptionKey.Generate();  // Will be overwritten
            var dekBytes = dataKey.Export();
            Buffer.BlockCopy(dataKeyBytes, 0, dekBytes, 0, dataKeyBytes.Length);
            
            var dataCrypto = new SecureCryptoService(dataKey);
            var plaintextResult = dataCrypto.Decrypt(envelope.EncryptedData);
            
            // Zero out key material
            Array.Clear(dataKeyBytes, 0, dataKeyBytes.Length);
            Array.Clear(dekBytes, 0, dekBytes.Length);
            
            return plaintextResult;
        }
        catch (Exception ex)
        {
            return Result<byte[], string>.Failure(
                $"Envelope decryption failed: {ex.Message}");
        }
    }
}
```

### Digital Signatures

```csharp
/// <summary>
/// Digital signature for data integrity and authenticity.
/// </summary>
public sealed record DigitalSignature
{
    public byte[] Signature { get; }
    public byte[] PublicKey { get; }
    
    internal DigitalSignature(byte[] signature, byte[] publicKey)
    {
        Signature = signature;
        PublicKey = publicKey;
    }
}

/// <summary>
/// Signing service using modern algorithms.
/// </summary>
public sealed class SigningService : IDisposable
{
    private readonly ECDsa _privateKey;
    private bool _disposed;
    
    private SigningService(ECDsa privateKey)
    {
        _privateKey = privateKey;
    }
    
    /// <summary>
    /// Generates a new signing key pair.
    /// </summary>
    public static SigningService Generate()
    {
        var ecdsa = ECDsa.Create(ECCurve.NamedCurves.nistP256);
        return new SigningService(ecdsa);
    }
    
    /// <summary>
    /// Signs data using ECDSA.
    /// </summary>
    public DigitalSignature Sign(byte[] data)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        
        var signature = _privateKey.SignData(
            data,
            HashAlgorithmName.SHA256);
        
        var publicKey = _privateKey.ExportSubjectPublicKeyInfo();
        
        return new DigitalSignature(signature, publicKey);
    }
    
    /// <summary>
    /// Verifies a signature.
    /// </summary>
    public static bool Verify(byte[] data, DigitalSignature signature)
    {
        try
        {
            using var ecdsa = ECDsa.Create();
            ecdsa.ImportSubjectPublicKeyInfo(signature.PublicKey, out _);
            
            return ecdsa.VerifyData(
                data,
                signature.Signature,
                HashAlgorithmName.SHA256);
        }
        catch
        {
            return false;
        }
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            _privateKey.Dispose();
            _disposed = true;
        }
    }
}
```

### Key Rotation

```csharp
/// <summary>
/// Manages encryption key rotation.
/// </summary>
public sealed class KeyRotationService
{
    public sealed record VersionedKey
    {
        public required int Version { get; init; }
        public required EncryptionKey Key { get; init; }
        public required DateTime CreatedAt { get; init; }
    }
    
    private readonly Dictionary<int, VersionedKey> _keys = new();
    private int _currentVersion;
    
    public void AddKey(VersionedKey key)
    {
        _keys[key.Version] = key;
        if (key.Version > _currentVersion)
            _currentVersion = key.Version;
    }
    
    public VersionedKey GetCurrentKey()
    {
        return _keys[_currentVersion];
    }
    
    public Result<VersionedKey, string> GetKey(int version)
    {
        if (!_keys.TryGetValue(version, out var key))
            return Result<VersionedKey, string>.Failure(
                $"Key version {version} not found");
        
        return Result<VersionedKey, string>.Success(key);
    }
    
    public sealed record VersionedEncryptedData
    {
        public required int KeyVersion { get; init; }
        public required EncryptedData Data { get; init; }
    }
    
    /// <summary>
    /// Encrypts with the current key version.
    /// </summary>
    public Result<VersionedEncryptedData, string> Encrypt(byte[] plaintext)
    {
        var currentKey = GetCurrentKey();
        var crypto = new SecureCryptoService(currentKey.Key);
        
        var result = crypto.Encrypt(plaintext);
        
        return result.Match(
            onSuccess: encrypted => Result<VersionedEncryptedData, string>.Success(
                new VersionedEncryptedData
                {
                    KeyVersion = currentKey.Version,
                    Data = encrypted
                }),
            onFailure: error => Result<VersionedEncryptedData, string>.Failure(error));
    }
    
    /// <summary>
    /// Decrypts using the versioned key.
    /// </summary>
    public Result<byte[], string> Decrypt(VersionedEncryptedData encrypted)
    {
        var keyResult = GetKey(encrypted.KeyVersion);
        
        if (!keyResult.IsSuccess)
            return Result<byte[], string>.Failure(keyResult.Error!);
        
        var crypto = new SecureCryptoService(keyResult.Value!.Key);
        return crypto.Decrypt(encrypted.Data);
    }
    
    /// <summary>
    /// Re-encrypts data with the current key.
    /// </summary>
    public Result<VersionedEncryptedData, string> ReEncrypt(
        VersionedEncryptedData encrypted)
    {
        // Decrypt with old key
        var plaintextResult = Decrypt(encrypted);
        
        if (!plaintextResult.IsSuccess)
            return Result<VersionedEncryptedData, string>.Failure(
                plaintextResult.Error!);
        
        // Encrypt with current key
        return Encrypt(plaintextResult.Value!);
    }
}
```

## Testing

```csharp
public class CryptographicTests
{
    [Fact]
    public void EncryptDecrypt_RoundTrip_Succeeds()
    {
        using var key = EncryptionKey.Generate();
        var crypto = new SecureCryptoService(key);
        
        var plaintext = Encoding.UTF8.GetBytes("Secret message");
        
        var encryptResult = crypto.Encrypt(plaintext);
        Assert.True(encryptResult.IsSuccess);
        
        var decryptResult = crypto.Decrypt(encryptResult.Value!);
        Assert.True(decryptResult.IsSuccess);
        
        Assert.Equal(plaintext, decryptResult.Value);
    }
    
    [Fact]
    public void PasswordHash_VerifyCorrectPassword_ReturnsTrue()
    {
        var password = "MySecurePassword123!";
        var hash = PasswordHash.Create(password);
        
        Assert.True(hash.Verify(password));
    }
    
    [Fact]
    public void PasswordHash_VerifyWrongPassword_ReturnsFalse()
    {
        var password = "MySecurePassword123!";
        var hash = PasswordHash.Create(password);
        
        Assert.False(hash.Verify("WrongPassword"));
    }
    
    [Fact]
    public void SecureToken_Generate_IsUnpredictable()
    {
        var token1 = SecureTokenGenerator.Generate();
        var token2 = SecureTokenGenerator.Generate();
        
        Assert.NotEqual(token1, token2);
        Assert.True(token1.Length > 0);
    }
    
    [Fact]
    public void DigitalSignature_VerifyValid_ReturnsTrue()
    {
        using var signer = SigningService.Generate();
        
        var data = Encoding.UTF8.GetBytes("Important message");
        var signature = signer.Sign(data);
        
        Assert.True(SigningService.Verify(data, signature));
    }
    
    [Fact]
    public void DigitalSignature_TamperedData_ReturnsFalse()
    {
        using var signer = SigningService.Generate();
        
        var data = Encoding.UTF8.GetBytes("Important message");
        var signature = signer.Sign(data);
        
        // Tamper with data
        data[0] ^= 1;
        
        Assert.False(SigningService.Verify(data, signature));
    }
}
```

## Why It's a Problem

1. **Data breaches**: Weak crypto enables decryption of sensitive data
2. **Hardcoded keys**: Keys in source code are compromised
3. **Timing attacks**: Non-constant-time comparison leaks information
4. **Weak algorithms**: MD5, SHA1, DES are broken
5. **Implementation errors**: Manual crypto is error-prone

## Symptoms

- Using `MD5`, `SHA1`, `DES`, or `TripleDES`
- Hardcoded encryption keys or passwords
- ECB cipher mode
- String comparison for password verification
- `Random` instead of `RandomNumberGenerator` for security tokens
- Manual padding or IV generation

## Benefits

- **Modern algorithms**: AES-GCM, PBKDF2, ECDSA
- **Secure key management**: Keys loaded from vaults, not hardcoded
- **Timing attack prevention**: Constant-time comparisons
- **Type safety**: `EncryptionKey` and `PasswordHash` types
- **Authenticated encryption**: GCM mode prevents tampering

## Trade-offs

- **Complexity**: More types and abstractions
- **Key management**: Requires vault/HSM infrastructure
- **Performance**: Strong algorithms are slower (intentionally)
- **Breaking changes**: Existing encrypted data needs migration

## See Also

- [Secret Types](./secret-types.md) — preventing key exposure
- [Validated Configuration](./validated-configuration.md) — loading keys securely
- [Input Sanitization](./input-sanitization.md) — validating cryptographic inputs
- [API Key Rotation](./api-key-rotation.md) — time-limited credentials
