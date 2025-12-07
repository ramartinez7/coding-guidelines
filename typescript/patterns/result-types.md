# Result Types

> Using exceptions or undefined for error handling—use explicit Result types to make errors type-safe and visible.

## Problem

Functions that can fail by throwing exceptions or returning `undefined` hide their failure modes from the type system. Callers must either read documentation or discover failure cases at runtime.

## Example

### ❌ Before

```typescript
function parseAge(input: string): number {
  const age = parseInt(input, 10);
  
  if (isNaN(age)) {
    throw new Error('Invalid age format');
  }
  
  if (age < 0 || age > 150) {
    throw new Error('Age out of valid range');
  }
  
  return age;
}

// Caller has no way to know this can throw
try {
  const age = parseAge(userInput);
  console.log(`Age: ${age}`);
} catch (error) {
  // What kind of error? Have to check error message strings
  console.error(error);
}
```

### ✅ After

```typescript
type Result<T, E> = 
  | { readonly success: true; readonly value: T }
  | { readonly success: false; readonly error: E };

type ParseAgeError = 
  | { readonly kind: 'invalid-format'; readonly input: string }
  | { readonly kind: 'out-of-range'; readonly value: number; readonly min: number; readonly max: number };

function parseAge(input: string): Result<number, ParseAgeError> {
  const age = parseInt(input, 10);
  
  if (isNaN(age)) {
    return {
      success: false,
      error: { kind: 'invalid-format', input }
    };
  }
  
  if (age < 0 || age > 150) {
    return {
      success: false,
      error: { kind: 'out-of-range', value: age, min: 0, max: 150 }
    };
  }
  
  return { success: true, value: age };
}

// Caller must handle both success and failure
const result = parseAge(userInput);

if (result.success) {
  console.log(`Age: ${result.value}`);
} else {
  // Type-safe error handling
  switch (result.error.kind) {
    case 'invalid-format':
      console.error(`Cannot parse "${result.error.input}" as a number`);
      break;
    
    case 'out-of-range':
      console.error(`Age ${result.error.value} must be between ${result.error.min} and ${result.error.max}`);
      break;
  }
}
```

## Why It's a Problem

1. **Hidden failures**: Exceptions don't appear in the function signature—callers don't know what can go wrong.

2. **Runtime surprises**: Uncaught exceptions crash the program; forgot to check for `undefined` causes `TypeError`.

3. **Vague errors**: Checking error message strings is brittle and error-prone.

4. **Lost type safety**: TypeScript can't help with exception handling—it's all runtime.

## Symptoms

- Functions that throw exceptions for business logic failures (not just programmer errors)
- Functions returning `T | undefined` where `undefined` means "failed somehow"
- Try-catch blocks in business logic code
- Error messages checked with string matching
- Documentation comments explaining what exceptions can be thrown

## Result Type Utilities

### Basic Result Type

```typescript
type Result<T, E> = Success<T> | Failure<E>;

interface Success<T> {
  readonly success: true;
  readonly value: T;
}

interface Failure<E> {
  readonly success: false;
  readonly error: E;
}

// Helper functions
const Result = {
  success: <T>(value: T): Result<T, never> => ({
    success: true,
    value
  }),
  
  failure: <E>(error: E): Result<never, E> => ({
    success: false,
    error
  })
};
```

### Mapping and Chaining

```typescript
type Result<T, E> = Success<T> | Failure<E>;

interface Success<T> {
  readonly success: true;
  readonly value: T;
}

interface Failure<E> {
  readonly success: false;
  readonly error: E;
}

const Result = {
  success: <T>(value: T): Result<T, never> => ({
    success: true,
    value
  }),
  
  failure: <E>(error: E): Result<never, E> => ({
    success: false,
    error
  }),
  
  map: <T, U, E>(
    result: Result<T, E>,
    fn: (value: T) => U
  ): Result<U, E> => {
    if (result.success) {
      return Result.success(fn(result.value));
    }
    return result;
  },
  
  flatMap: <T, U, E>(
    result: Result<T, E>,
    fn: (value: T) => Result<U, E>
  ): Result<U, E> => {
    if (result.success) {
      return fn(result.value);
    }
    return result;
  },
  
  mapError: <T, E, F>(
    result: Result<T, E>,
    fn: (error: E) => F
  ): Result<T, F> => {
    if (!result.success) {
      return Result.failure(fn(result.error));
    }
    return result;
  },
  
  unwrapOr: <T, E>(result: Result<T, E>, defaultValue: T): T => {
    return result.success ? result.value : defaultValue;
  }
};
```

### Example: Chaining Multiple Operations

```typescript
interface User {
  id: string;
  email: string;
}

type LoadUserError = 
  | { readonly kind: 'not-found'; readonly id: string }
  | { readonly kind: 'database-error'; readonly message: string };

type ValidateEmailError = 
  | { readonly kind: 'invalid-format'; readonly email: string }
  | { readonly kind: 'domain-blocked'; readonly domain: string };

function loadUser(id: string): Result<User, LoadUserError> {
  // ... load from database
}

function validateEmail(email: string): Result<string, ValidateEmailError> {
  // ... validate email
}

function sendWelcomeEmail(email: string): Result<void, string> {
  // ... send email
}

// Chain operations together
const result = Result.flatMap(
  loadUser('user-123'),
  user => Result.flatMap(
    validateEmail(user.email),
    validEmail => sendWelcomeEmail(validEmail)
  )
);

if (!result.success) {
  // Handle any error in the chain
  console.error('Failed to send welcome email:', result.error);
}
```

## Async Results

For operations that return Promises:

```typescript
type AsyncResult<T, E> = Promise<Result<T, E>>;

async function loadUserAsync(id: string): AsyncResult<User, LoadUserError> {
  try {
    const user = await database.users.findById(id);
    
    if (!user) {
      return Result.failure({ kind: 'not-found', id });
    }
    
    return Result.success(user);
  } catch (error) {
    return Result.failure({ 
      kind: 'database-error', 
      message: error instanceof Error ? error.message : 'Unknown error' 
    });
  }
}

// Usage
const result = await loadUserAsync('user-123');

if (result.success) {
  console.log('User:', result.value);
} else {
  console.error('Failed to load user:', result.error);
}
```

## When to Use Exceptions vs Results

**Use Results for:**
- Expected failures (validation errors, not found, business rule violations)
- Recoverable errors that callers should handle
- Errors that are part of the domain (user already exists, insufficient funds)

**Use Exceptions for:**
- Programmer errors (null reference, array out of bounds)
- Unrecoverable errors (out of memory, stack overflow)
- Third-party library errors that you can't reasonably handle
- Truly exceptional circumstances

## Benefits

- **Explicit error handling**: Function signature shows all possible outcomes
- **Type-safe errors**: Discriminated unions make error types explicit and exhaustive
- **No silent failures**: Compiler forces you to handle both success and error cases
- **Better composition**: Results can be mapped and chained functionally
- **Self-documenting**: The return type tells you exactly what can go wrong

## See Also

- [Honest Functions](./honest-functions.md)
- [Discriminated Unions](./discriminated-unions.md)
- [Value Semantics](./value-semantics.md)
