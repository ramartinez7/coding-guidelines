# Strongly Typed IDs

> Multiple ID parameters of the same primitive type—wrap each in a distinct type to prevent swapping.

## Problem

When functions accept multiple IDs of the same underlying type (`string`, `number`), the compiler can't distinguish between them. Swapped arguments compile successfully but fail catastrophically at runtime.

## Example

### ❌ Before

```typescript
class OrderService {
  // All three are strings—compiler treats them identically
  addItem(userId: string, orderId: string, productId: string): void {
    // ...
  }
}

// Caller
const uId = generateId();
const oId = generateId();
const pId = generateId();

// Compiles perfectly, fails catastrophically at runtime
service.addItem(oId, uId, pId);  // Wrong order! But no compiler error.
```

### ✅ After

```typescript
// Branded type approach
type UserId = string & { readonly __brand: 'UserId' };
type OrderId = string & { readonly __brand: 'OrderId' };
type ProductId = string & { readonly __brand: 'ProductId' };

// Factory functions to create branded types
const UserId = {
  create: (value: string): UserId => value as UserId,
  new: (): UserId => generateId() as UserId,
};

const OrderId = {
  create: (value: string): OrderId => value as OrderId,
  new: (): OrderId => generateId() as OrderId,
};

const ProductId = {
  create: (value: string): ProductId => value as ProductId,
  new: (): ProductId => generateId() as ProductId,
};

class OrderService {
  // Each parameter is now a distinct type
  addItem(userId: UserId, orderId: OrderId, productId: ProductId): void {
    // ...
  }
}

// Caller
const uId = UserId.new();
const oId = OrderId.new();
const pId = ProductId.new();

// Compiler error: Type 'OrderId' is not assignable to type 'UserId'
service.addItem(oId, uId, pId);  // ❌ Won't compile!

// Correct usage
service.addItem(uId, oId, pId);  // ✅ Types enforce correct order
```

### Alternative: Class-Based Approach

```typescript
class UserId {
  private constructor(readonly value: string) {}
  
  static create(value: string): UserId {
    return new UserId(value);
  }
  
  static new(): UserId {
    return new UserId(generateId());
  }
  
  toString(): string {
    return this.value;
  }
}

class OrderId {
  private constructor(readonly value: string) {}
  
  static create(value: string): OrderId {
    return new OrderId(value);
  }
  
  static new(): OrderId {
    return new OrderId(generateId());
  }
  
  toString(): string {
    return this.value;
  }
}

class ProductId {
  private constructor(readonly value: string) {}
  
  static create(value: string): ProductId {
    return new ProductId(value);
  }
  
  static new(): ProductId {
    return new ProductId(generateId());
  }
  
  toString(): string {
    return this.value;
  }
}

class OrderService {
  addItem(userId: UserId, orderId: OrderId, productId: ProductId): void {
    // Access the underlying value with .value
    console.log(`Adding item ${productId.value} to order ${orderId.value} for user ${userId.value}`);
  }
}
```

## Why It's a Problem

1. **Argument swapping**: The most common source of subtle bugs—everything compiles, tests might even pass with mock data, but production fails.

2. **Loss of intent**: A `string` is an implementation detail; a `CustomerId` is a domain concept with meaning.

3. **No IDE help**: Autocomplete suggests any `string` variable for any `string` parameter.

4. **Refactoring hazards**: Reordering parameters breaks callers silently.

## Symptoms

- Functions with multiple `string`, `number` parameters representing different IDs
- Bugs discovered in production where "the wrong ID was passed"
- Comments like `// userId, not orderId!`
- Variable naming conventions to compensate (`customerIdStr`, `orderIdStr`)
- Unit tests that don't catch ID mix-ups because all test IDs are similar

## Branded Types vs Classes

### Branded Types (Nominal Typing)

**Pros:**
- Zero runtime overhead—just TypeScript compile-time checking
- Works with existing serialization (JSON, databases) without conversion
- Lightweight syntax

**Cons:**
- Requires type assertions (`as UserId`)
- Less discoverable—factory functions needed for safety
- Can be bypassed with type assertions

### Class-Based Types

**Pros:**
- Runtime enforcement—impossible to bypass
- More discoverable—constructors and methods clearly visible
- Better for complex value objects with behavior
- Can include validation logic

**Cons:**
- Runtime overhead (minimal but present)
- Requires conversion for serialization (`.value` property)
- More boilerplate

**Recommendation:** Use branded types for simple IDs, classes for value objects with validation or behavior.

## JSON Serialization

### For Branded Types

```typescript
// Serialize
const json = JSON.stringify({ userId: uId });  // Works automatically

// Deserialize with validation
const data = JSON.parse(json);
const userId = UserId.create(data.userId);
```

### For Class-Based Types

```typescript
// Serialize
const json = JSON.stringify({ userId: userId.value });

// Deserialize
const data = JSON.parse(json);
const userId = UserId.create(data.userId);
```

## Benefits

- **Compile-time safety**: Swapped arguments don't compile
- **Self-documenting**: `addItem(userId: UserId, orderId: OrderId, productId: ProductId)` is unambiguous
- **Refactoring-safe**: Reordering parameters causes compiler errors at all call sites
- **IDE support**: Autocomplete only suggests variables of the correct type
- **Domain modeling**: IDs become first-class domain concepts

## See Also

- [Primitive Obsession](./primitive-obsession.md)
- [Value Semantics](./value-semantics.md)
- [Static Factory Methods](./static-factory-methods.md)
- [Branded Types](./branded-types.md)
