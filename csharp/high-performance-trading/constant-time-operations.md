# Constant-Time Operations

> Guarantee timing-independent of input data for security-critical code—prevent timing attacks by ensuring operations take the same time regardless of values.

## Problem

Variable-time operations leak information through timing side-channels. In security-critical systems (authentication, cryptography, trading algorithms), attackers can measure execution time to infer secret data. Traditional optimizations like early returns, branches, and lookups create timing variations that compromise security. Constant-time operations eliminate these channels.

## Example

### ❌ Before (Variable-Time, Vulnerable)

```csharp
public class VariableTimeAuth
{
    // ❌ Timing attack vulnerable!
    public bool VerifyPassword(string password, string storedHash)
    {
        string computedHash = ComputeHash(password);
        
        // String comparison returns early on first mismatch
        // Attacker can measure time to guess each character!
        return computedHash == storedHash;
    }
    
    // ❌ Branch reveals information
    public bool CheckPermission(int userId, int requiredPermission)
    {
        if (userId == AdminUserId)
            return true;  // Returns immediately for admin
        
        return HasPermission(userId, requiredPermission);  // Slower for others
        
        // Timing reveals if userId is admin!
    }
    
    // ❌ Lookup time varies by key
    public decimal GetPrice(string symbol, Dictionary<string, decimal> prices)
    {
        if (prices.ContainsKey(symbol))
            return prices[symbol];
        
        return 0;
        
        // Time difference reveals if symbol exists
    }
    
    private string ComputeHash(string password) => "";
    private const int AdminUserId = 0;
    private bool HasPermission(int userId, int requiredPermission) => false;
}
```

### ✅ After (Constant-Time, Secure)

```csharp
public class ConstantTimeAuth
{
    // ✅ Constant-time string comparison
    public bool VerifyPassword(string password, string storedHash)
    {
        byte[] computed = Encoding.UTF8.GetBytes(ComputeHash(password));
        byte[] stored = Encoding.UTF8.GetBytes(storedHash);
        
        return ConstantTimeEquals(computed, stored);
    }
    
    // ✅ Constant-time byte comparison
    private static bool ConstantTimeEquals(byte[] a, byte[] b)
    {
        if (a.Length != b.Length)
        {
            // Comparison must still take constant time!
            b = new byte[a.Length];
        }
        
        uint result = 0;
        
        // Always compares all bytes
        for (int i = 0; i < a.Length; i++)
        {
            result |= (uint)(a[i] ^ b[i]);
        }
        
        return result == 0;
    }
    
    // ✅ Constant-time permission check
    public bool CheckPermission(int userId, int requiredPermission)
    {
        bool isAdmin = ConstantTimeEquals(userId, AdminUserId);
        bool hasPermission = HasPermission(userId, requiredPermission);
        
        // Both branches evaluated, result combined
        return isAdmin | hasPermission;  // Bitwise OR (not short-circuit)
    }
    
    private static bool ConstantTimeEquals(int a, int b)
    {
        // XOR: 0 if equal, non-zero if different
        int diff = a ^ b;
        
        // Convert to boolean in constant time
        // -diff has MSB set iff diff != 0
        // Shift MSB to LSB and negate
        return ((diff | -diff) >> 31) == 0;
    }
    
    private string ComputeHash(string password) => "";
    private const int AdminUserId = 0;
    private bool HasPermission(int userId, int requiredPermission) => false;
}
```

## Constant-Time Selection

```csharp
public static class ConstantTimeOps
{
    // ✅ Constant-time conditional move
    public static int Select(bool condition, int ifTrue, int ifFalse)
    {
        // Convert bool to mask: 0 or -1
        int mask = condition ? -1 : 0;
        
        // Bitwise selection (no branches)
        return (ifTrue & mask) | (ifFalse & ~mask);
    }
    
    // ✅ Constant-time min/max
    public static int Min(int a, int b)
    {
        int diff = a - b;
        int mask = diff >> 31;  // -1 if a < b, else 0
        return b + (diff & mask);
    }
    
    public static int Max(int a, int b)
    {
        int diff = a - b;
        int mask = diff >> 31;
        return a - (diff & mask);
    }
    
    // ✅ Constant-time absolute value
    public static int Abs(int value)
    {
        int mask = value >> 31;  // -1 if negative, 0 if positive
        return (value + mask) ^ mask;
    }
    
    // ✅ Constant-time sign
    public static int Sign(int value)
    {
        return (value >> 31) | (int)((uint)-value >> 31);
    }
}
```

## Constant-Time Array Operations

```csharp
public static class ConstantTimeArray
{
    // ✅ Constant-time array search (always scans all)
    public static int FindIndexConstantTime(int[] array, int target)
    {
        int result = -1;
        
        for (int i = 0; i < array.Length; i++)
        {
            // Compare without branching
            int isMatch = ((array[i] ^ target) == 0) ? 1 : 0;
            int mask = -isMatch;  // 0 or -1
            
            // Update result only if match (but always evaluate)
            result = (i & mask) | (result & ~mask);
        }
        
        return result;
    }
    
    // ✅ Constant-time array copy (with obfuscation)
    public static void SecureCopy(byte[] source, byte[] dest, int length)
    {
        // Always copy full length to hide actual size
        int copyLength = Math.Max(length, Math.Min(source.Length, dest.Length));
        
        for (int i = 0; i < copyLength; i++)
        {
            byte value = i < length ? source[i] : (byte)0;
            dest[i] = value;
        }
    }
    
    // ✅ Constant-time array clear
    public static void SecureClear(byte[] array)
    {
        // Prevent compiler optimization
        for (int i = 0; i < array.Length; i++)
        {
            array[i] = 0;
        }
        
        // Force write to memory
        System.GC.KeepAlive(array);
    }
}
```

## Constant-Time Comparison

```csharp
public static class ConstantTimeCompare
{
    // ✅ Constant-time integer comparison
    public static bool Equals(long a, long b)
    {
        long diff = a ^ b;
        
        // Reduce to single bit
        diff |= diff >> 32;
        diff |= diff >> 16;
        diff |= diff >> 8;
        diff |= diff >> 4;
        diff |= diff >> 2;
        diff |= diff >> 1;
        
        return (diff & 1) == 0;
    }
    
    // ✅ Constant-time string comparison
    public static bool Equals(string a, string b)
    {
        if (a == null || b == null)
        {
            return a == b;
        }
        
        // Pad to same length to hide actual lengths
        int maxLen = Math.Max(a.Length, b.Length);
        
        int result = a.Length ^ b.Length;  // Include length in comparison
        
        for (int i = 0; i < maxLen; i++)
        {
            char charA = i < a.Length ? a[i] : '\0';
            char charB = i < b.Length ? b[i] : '\0';
            result |= charA ^ charB;
        }
        
        return result == 0;
    }
    
    // ✅ Constant-time memory comparison
    public static unsafe bool Equals(byte* a, byte* b, int length)
    {
        uint result = 0;
        
        for (int i = 0; i < length; i++)
        {
            result |= (uint)(a[i] ^ b[i]);
        }
        
        return result == 0;
    }
}
```

## Constant-Time Cryptographic Operations

```csharp
public class ConstantTimeCrypto
{
    // ✅ Constant-time MAC verification
    public bool VerifyMAC(byte[] message, byte[] expectedMAC, byte[] key)
    {
        byte[] computedMAC = ComputeMAC(message, key);
        
        // Constant-time comparison
        return ConstantTimeArray.SequenceEqual(computedMAC, expectedMAC);
    }
    
    // ✅ Constant-time key comparison
    public bool VerifyKey(byte[] providedKey, byte[] storedKey)
    {
        if (providedKey.Length != storedKey.Length)
        {
            // Still take time proportional to stored key
            providedKey = new byte[storedKey.Length];
        }
        
        uint result = 0;
        
        for (int i = 0; i < storedKey.Length; i++)
        {
            result |= (uint)(providedKey[i] ^ storedKey[i]);
        }
        
        return result == 0;
    }
    
    // ✅ Constant-time signature verification
    public bool VerifySignature(byte[] message, byte[] signature, byte[] publicKey)
    {
        bool valid = VerifySignatureInternal(message, signature, publicKey);
        
        // Add noise to timing to prevent attacks
        Thread.SpinWait(RandomTiming());
        
        return valid;
    }
    
    private byte[] ComputeMAC(byte[] message, byte[] key) => new byte[32];
    private bool VerifySignatureInternal(byte[] m, byte[] s, byte[] k) => true;
    private int RandomTiming() => new Random().Next(100, 200);
}

public static class ConstantTimeArrayExtensions
{
    public static bool SequenceEqual(byte[] a, byte[] b)
    {
        if (a.Length != b.Length)
            return false;
        
        uint result = 0;
        
        for (int i = 0; i < a.Length; i++)
        {
            result |= (uint)(a[i] ^ b[i]);
        }
        
        return result == 0;
    }
}
```

## Trading Algorithm Protection

```csharp
public class ProtectedTradingAlgorithm
{
    // ✅ Constant-time order validation
    public bool ValidateOrder(decimal price, int quantity)
    {
        // All checks take same time
        bool priceValid = ConstantTimeCompare(price, MinPrice, MaxPrice);
        bool quantityValid = ConstantTimeCompare(quantity, MinQuantity, MaxQuantity);
        
        // Combine results (no short-circuit)
        return priceValid & quantityValid;
    }
    
    private static bool ConstantTimeCompare(decimal value, decimal min, decimal max)
    {
        // Convert to fixed-point
        long valueLong = (long)(value * 10000);
        long minLong = (long)(min * 10000);
        long maxLong = (long)(max * 10000);
        
        // Compute both comparisons
        int aboveMin = (int)((valueLong - minLong) >> 63);  // 0 if >=
        int belowMax = (int)((maxLong - valueLong) >> 63);  // 0 if <=
        
        // Both must be 0 (valid)
        return (aboveMin | belowMax) == 0;
    }
    
    // ✅ Constant-time symbol lookup
    public decimal GetPrice(string symbol, Dictionary<string, decimal> prices)
    {
        decimal result = 0;
        
        // Iterate all entries (no early exit)
        foreach (var kvp in prices)
        {
            bool match = ConstantTimeCompare.Equals(kvp.Key, symbol);
            int mask = match ? -1 : 0;
            
            // Select value if match (but always evaluate)
            long priceLong = (long)(kvp.Value * 10000);
            long resultLong = (long)(result * 10000);
            resultLong = (priceLong & mask) | (resultLong & ~mask);
            result = resultLong / 10000m;
        }
        
        return result;
    }
    
    private const decimal MinPrice = 0.01m;
    private const decimal MaxPrice = 100000m;
    private const int MinQuantity = 1;
    private const int MaxQuantity = 1000000;
}
```

## Padding and Noise

```csharp
public class TimingNoise
{
    // ✅ Add random timing noise
    public bool AuthenticateWithNoise(string username, string password)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        
        bool result = Authenticate(username, password);
        
        sw.Stop();
        
        // Pad to minimum time (e.g., 10ms)
        long minTicks = TimeSpan.FromMilliseconds(10).Ticks;
        long remaining = minTicks - sw.Elapsed.Ticks;
        
        if (remaining > 0)
        {
            Thread.Sleep(TimeSpan.FromTicks(remaining));
        }
        
        // Add random jitter
        Thread.SpinWait(Random.Shared.Next(100, 1000));
        
        return result;
    }
    
    private bool Authenticate(string username, string password) => true;
}
```

## Best Practices

### 1. No Early Returns

```csharp
// ❌ Early return leaks timing
public bool Validate(int[] data)
{
    for (int i = 0; i < data.Length; i++)
    {
        if (data[i] < 0)
            return false;  // Timing reveals position of first negative
    }
    return true;
}

// ✅ Always scan entire array
public bool Validate(int[] data)
{
    int result = 0;
    
    for (int i = 0; i < data.Length; i++)
    {
        result |= data[i] >> 31;  // Accumulate sign bits
    }
    
    return result == 0;
}
```

### 2. No Data-Dependent Branches

```csharp
// ❌ Branch depends on data
int result = value > 0 ? positive : negative;

// ✅ Branchless selection
int mask = (value >> 31);
int result = (positive & ~mask) | (negative & mask);
```

### 3. Use Bitwise Operators (Not Logical)

```csharp
// ❌ Logical operators short-circuit
bool result = condition1 && condition2;

// ✅ Bitwise operators evaluate both
bool result = condition1 & condition2;
```

### 4. Clear Sensitive Data

```csharp
// ✅ Clear password/key after use
byte[] password = GetPassword();
try
{
    UsePassword(password);
}
finally
{
    ConstantTimeArray.SecureClear(password);
}
```

## Performance Characteristics

- **Security**: Eliminates timing side-channels
- **Speed**: 10-50% slower than variable-time
- **Predictability**: Execution time independent of data
- **Attack resistance**: Prevents timing analysis

## When to Use

- Cryptographic operations (MAC, signature verification)
- Authentication and authorization
- Security-sensitive comparisons
- Proprietary trading algorithms
- Payment processing
- Password verification

## When NOT to Use

- Non-security-critical code
- Performance more important than security
- Data not sensitive
- Timing already visible (e.g., network latency)

## Related Patterns

- [Branchless Programming](./branchless-programming.md) — Branchless techniques
- [Bit Manipulation](./bit-manipulation-techniques.md) — Bitwise operations
- [Capability Security](../patterns/capability-security-authorization.md) — Security patterns

## References

- "Timing Attacks on Implementations of Diffie-Hellman, RSA, DSS" by Kocher
- "Cache-Timing Attacks on AES" by Bernstein
- "Constant-Time Cryptography" - BearSSL documentation
- "A note on constant-time comparisons" - NaCl documentation
