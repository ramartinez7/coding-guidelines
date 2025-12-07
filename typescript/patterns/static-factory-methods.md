# Static Factory Methods

> Use static methods instead of constructors to create instances with clear, descriptive names.

## Problem

Constructors have limitations: they must be named after the class, can't have different names for different creation scenarios, and can't conditionally return different types. Static factory methods overcome these limitations.

## Example

### ❌ Before

```typescript
class Money {
  constructor(
    readonly amount: number,
    readonly currency: string
  ) {
    if (amount < 0) {
      throw new Error('Amount cannot be negative');
    }
  }
}

// Ambiguous: what currency is this?
const price = new Money(19.99, 'USD');

// Multiple constructors not possible in TypeScript
// How do we create from cents? From another currency?
```

### ✅ After

```typescript
class Money {
  private constructor(
    readonly amount: number,
    readonly currency: string
  ) {}

  static usd(amount: number): Money {
    if (amount < 0) {
      throw new Error('Amount cannot be negative');
    }
    return new Money(amount, 'USD');
  }

  static eur(amount: number): Money {
    if (amount < 0) {
      throw new Error('Amount cannot be negative');
    }
    return new Money(amount, 'EUR');
  }

  static fromCents(cents: number, currency: string): Money {
    return new Money(cents / 100, currency);
  }

  static zero(currency: string): Money {
    return new Money(0, currency);
  }

  toString(): string {
    return `${this.amount.toFixed(2)} ${this.currency}`;
  }
}

// Clear and self-documenting
const price = Money.usd(19.99);
const fee = Money.fromCents(250, 'USD');  // $2.50
const starting = Money.zero('USD');
```

## Why It's a Problem

1. **Unclear intent**: Constructors can't communicate *why* or *how* you're creating an object.

2. **No overloading**: TypeScript doesn't support multiple constructors with different signatures.

3. **No conditional creation**: Constructors must return an instance; can't return null or a different type.

4. **No caching**: Every `new` call creates a new instance; can't reuse immutable instances.

## Symptoms

- Constructors with boolean flags to control behavior
- Comments explaining what constructor parameters mean
- Multiple ways to represent the same concept (dollars vs cents, seconds vs milliseconds)
- Constructor parameter lists that are ambiguous
- Validation logic scattered across multiple constructor calls

## Benefits of Static Factory Methods

### 1. Descriptive Names

```typescript
class DateRange {
  private constructor(
    readonly start: Date,
    readonly end: Date
  ) {}

  static fromDates(start: Date, end: Date): DateRange {
    return new DateRange(start, end);
  }

  static lastNDays(days: number): DateRange {
    const end = new Date();
    const start = new Date();
    start.setDate(start.getDate() - days);
    return new DateRange(start, end);
  }

  static thisMonth(): DateRange {
    const now = new Date();
    const start = new Date(now.getFullYear(), now.getMonth(), 1);
    const end = new Date(now.getFullYear(), now.getMonth() + 1, 0);
    return new DateRange(start, end);
  }

  static thisYear(): DateRange {
    const now = new Date();
    const start = new Date(now.getFullYear(), 0, 1);
    const end = new Date(now.getFullYear(), 11, 31);
    return new DateRange(start, end);
  }
}

// Self-documenting usage
const last7Days = DateRange.lastNDays(7);
const currentMonth = DateRange.thisMonth();
```

### 2. Validation and Parsing

```typescript
class EmailAddress {
  private constructor(readonly value: string) {}

  static fromString(input: string): EmailAddress {
    const normalized = input.trim().toLowerCase();
    
    if (!normalized.includes('@')) {
      throw new Error(`Invalid email format: ${input}`);
    }
    
    return new EmailAddress(normalized);
  }

  static tryParse(input: string): EmailAddress | undefined {
    try {
      return EmailAddress.fromString(input);
    } catch {
      return undefined;
    }
  }

  toString(): string {
    return this.value;
  }
}

// Clear intent
const email = EmailAddress.fromString('user@example.com');  // Throws on invalid

const maybeEmail = EmailAddress.tryParse(userInput);  // Returns undefined on invalid
if (maybeEmail) {
  console.log('Valid email:', maybeEmail.toString());
}
```

### 3. Returning Different Types

```typescript
interface Success<T> {
  readonly success: true;
  readonly value: T;
}

interface Failure<E> {
  readonly success: false;
  readonly error: E;
}

type Result<T, E> = Success<T> | Failure<E>;

class Result {
  private constructor() {}  // Never called directly

  static success<T>(value: T): Result<T, never> {
    return { success: true, value };
  }

  static failure<E>(error: E): Result<never, E> {
    return { success: false, error };
  }
}

// Factory methods return different shapes
const ok = Result.success(42);
const err = Result.failure('Something went wrong');
```

### 4. Caching and Reusing Instances

```typescript
class Status {
  private static readonly ACTIVE = new Status('active');
  private static readonly INACTIVE = new Status('inactive');
  private static readonly PENDING = new Status('pending');

  private constructor(readonly value: string) {}

  static active(): Status {
    return Status.ACTIVE;  // Always returns the same instance
  }

  static inactive(): Status {
    return Status.INACTIVE;
  }

  static pending(): Status {
    return Status.PENDING;
  }
}

// All references to active() point to the same object
const status1 = Status.active();
const status2 = Status.active();
console.log(status1 === status2);  // true (same reference)
```

### 5. Multiple Representations

```typescript
class Temperature {
  private constructor(readonly kelvin: number) {}

  static fromCelsius(celsius: number): Temperature {
    return new Temperature(celsius + 273.15);
  }

  static fromFahrenheit(fahrenheit: number): Temperature {
    return new Temperature((fahrenheit - 32) * 5/9 + 273.15);
  }

  static fromKelvin(kelvin: number): Temperature {
    return new Temperature(kelvin);
  }

  toCelsius(): number {
    return this.kelvin - 273.15;
  }

  toFahrenheit(): number {
    return (this.kelvin - 273.15) * 9/5 + 32;
  }

  toKelvin(): number {
    return this.kelvin;
  }
}

// Clear which unit you're using
const temp1 = Temperature.fromCelsius(25);
const temp2 = Temperature.fromFahrenheit(77);
console.log(temp1.toFahrenheit());  // 77
```

## Common Patterns

### Named Constructors

```typescript
class User {
  private constructor(
    readonly id: string,
    readonly email: string,
    readonly role: string
  ) {}

  static create(email: string): User {
    return new User(generateId(), email, 'user');
  }

  static createAdmin(email: string): User {
    return new User(generateId(), email, 'admin');
  }
}
```

### From Existing Data

```typescript
class Product {
  private constructor(
    readonly id: string,
    readonly name: string,
    readonly price: Money
  ) {}

  static fromJson(json: any): Product {
    return new Product(
      json.id,
      json.name,
      Money.fromCents(json.priceCents, json.currency)
    );
  }

  static fromDatabase(row: DbRow): Product {
    return new Product(
      row.product_id,
      row.product_name,
      Money.fromCents(row.price_cents, row.currency_code)
    );
  }
}
```

## When to Use

**Use static factory methods when:**
- You need multiple ways to create an object
- Creation involves validation or normalization
- The creation process is complex or non-obvious
- You want to cache or reuse instances
- Different creation paths need different validation rules

**Use constructors when:**
- Simple, straightforward object creation
- All parameters are required and validated by the type system
- No alternative construction methods needed

## Benefits

- **Self-documenting**: Method names describe intent
- **Flexible**: Can return different types, cache instances, or fail gracefully
- **Maintainable**: Centralize creation logic in one place
- **Type-safe**: Combine with private constructors to enforce validation

## See Also

- [Primitive Obsession](./primitive-obsession.md)
- [Value Semantics](./value-semantics.md)
- [Result Types](./result-types.md)
