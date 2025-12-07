# Primitive Obsession

> Over-use of "primitive" built-in types instead of domain-specific types.

## Problem

Using primitives like `string` or `number` for domain concepts loses meaning and safety. Validation logic gets scattered and duplicated, call sites become ambiguous, and rule changes require codebase-wide edits.

## Example

### ❌ Before

```typescript
class UserService {
  registerUser(email: string, displayName: string): void {
    if (!email || !email.includes('@')) {
      throw new Error('Invalid email');
    }

    console.log(`Welcome ${displayName}! We'll email you at ${email.trim().toLowerCase()}.`);
  }
}
```

### ✅ After

```typescript
class EmailAddress {
  private constructor(private readonly value: string) {}

  static fromString(value: string): EmailAddress {
    if (!value || !value.includes('@')) {
      throw new Error('Invalid email');
    }
    return new EmailAddress(value.trim().toLowerCase());
  }

  static tryFromString(value: string): EmailAddress | undefined {
    if (!value || !value.includes('@')) {
      return undefined;
    }
    return new EmailAddress(value.trim().toLowerCase());
  }

  static create(value: string): Result<EmailAddress, string> {
    if (!value || !value.includes('@')) {
      return Result.failure('Invalid email format');
    }
    return Result.success(new EmailAddress(value.trim().toLowerCase()));
  }

  toString(): string {
    return this.value;
  }
}

class UserService {
  registerUser(email: EmailAddress, displayName: string): void {
    console.log(`Welcome ${displayName}! We'll email you at ${email.toString()}.`);
  }
}

// Usage:
const email = EmailAddress.fromString('user@example.com');  // Throws if invalid

const validEmail = EmailAddress.tryFromString(userInput);  // Returns undefined if invalid
if (validEmail) {
  service.registerUser(validEmail, 'Alice');
}

const result = EmailAddress.create(userInput);  // Result pattern
if (result.isSuccess) {
  service.registerUser(result.value, 'Bob');
} else {
  console.warn('Registration failed:', result.error);
}
```

## Why It's a Problem

1. **Lost domain meaning**: Any string looks valid to the compiler; runtime accepts garbage that passes naive checks.
2. **Scattered invariants**: Validation/normalization repeated inconsistently across modules.
3. **Ambiguity**: Call sites don't communicate intent—is this a username, ID, or email?
4. **Maintenance cost**: Rule changes require codebase-wide edits (easy to miss one).

## Symptoms

- Built-in types (`string`, `number`, etc.) in function signatures where domain types would be more appropriate
- Copy/pasted value validation across multiple locations
- Value validation outside of a constructor or factory method
- Asking yourself: Do I really need a string with unlimited length? Negative numbers? Arbitrary precision?

## Benefits

- **Meaningful types** communicate intent clearly
- **Single source of truth** for validation and formatting
- **Safer APIs** prevent misuse at compile time

## See Also

- [Data Clumps](./data-clump.md)
- [Static Factory Methods](./static-factory-methods.md)
- [Strongly Typed IDs](./strongly-typed-ids.md)
