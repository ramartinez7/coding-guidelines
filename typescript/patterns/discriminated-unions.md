# Discriminated Unions

> Using multiple optional properties where only certain combinations are valid—model variants as a discriminated union instead.

## Problem

When an interface has multiple optional properties where only specific combinations make sense, it's easy to create invalid states. Discriminated unions (also called tagged unions or sum types) make illegal states unrepresentable.

## Example

### ❌ Before

```typescript
interface Order {
  id: string;
  status: 'pending' | 'shipped' | 'delivered';
  shippedAt?: Date;
  trackingNumber?: string;
  deliveredAt?: Date;
  signature?: string;
}

function processOrder(order: Order): void {
  if (order.status === 'shipped') {
    // Can we trust that trackingNumber exists?
    console.log(`Order shipped with tracking: ${order.trackingNumber}`);
  }
  
  if (order.status === 'delivered') {
    // Can we trust that deliveredAt exists?
    console.log(`Order delivered at: ${order.deliveredAt}`);
  }
}

// ❌ Invalid states are representable
const invalidOrder: Order = {
  id: '123',
  status: 'pending',
  shippedAt: new Date(),  // Shouldn't have shippedAt if pending!
  trackingNumber: 'TRACK123'  // Shouldn't have tracking if pending!
};
```

### ✅ After

```typescript
interface PendingOrder {
  readonly kind: 'pending';
  readonly id: string;
}

interface ShippedOrder {
  readonly kind: 'shipped';
  readonly id: string;
  readonly shippedAt: Date;
  readonly trackingNumber: string;
}

interface DeliveredOrder {
  readonly kind: 'delivered';
  readonly id: string;
  readonly shippedAt: Date;
  readonly trackingNumber: string;
  readonly deliveredAt: Date;
  readonly signature: string;
}

type Order = PendingOrder | ShippedOrder | DeliveredOrder;

function processOrder(order: Order): void {
  switch (order.kind) {
    case 'pending':
      console.log(`Order ${order.id} is pending`);
      break;
    
    case 'shipped':
      // TypeScript knows trackingNumber exists here
      console.log(`Order shipped with tracking: ${order.trackingNumber}`);
      break;
    
    case 'delivered':
      // TypeScript knows all delivery fields exist here
      console.log(`Order delivered at: ${order.deliveredAt}`);
      break;
    
    // TypeScript enforces exhaustiveness—error if we forget a case
  }
}

// ✅ Invalid states are now impossible to represent
const pendingOrder: Order = {
  kind: 'pending',
  id: '123'
  // Can't add shippedAt or trackingNumber—they don't exist on PendingOrder
};

const shippedOrder: Order = {
  kind: 'shipped',
  id: '123',
  shippedAt: new Date(),
  trackingNumber: 'TRACK123'
  // Must provide both shippedAt and trackingNumber
};
```

## Why It's a Problem

1. **Invalid states**: Optional properties allow nonsensical combinations (pending order with tracking number).

2. **Runtime checks**: Code must defensively check existence of optional fields, even when the state implies they should exist.

3. **No exhaustiveness**: Adding a new state requires manually finding all places that handle states—easy to miss one.

4. **Lost guarantees**: When you know the state, you should *know* which fields are available, not guess.

## Symptoms

- Multiple optional properties on an interface where certain combinations are invalid
- Comments like `// Only set when status is 'shipped'`
- Runtime checks like `if (order.status === 'shipped' && order.trackingNumber)`
- TypeScript warnings about possibly undefined properties even when you "know" they exist
- State machine behavior implemented with strings and optional fields

## TypeScript Features

### Discriminant Property

The `kind` property (also called a "tag" or "discriminant") allows TypeScript to narrow the type:

```typescript
function getTracking(order: Order): string | undefined {
  if (order.kind === 'shipped' || order.kind === 'delivered') {
    // TypeScript knows trackingNumber exists on both shipped and delivered
    return order.trackingNumber;
  }
  return undefined;
}
```

### Exhaustiveness Checking

TypeScript can enforce that all cases are handled:

```typescript
function processOrder(order: Order): void {
  switch (order.kind) {
    case 'pending':
      return handlePending(order);
    case 'shipped':
      return handleShipped(order);
    case 'delivered':
      return handleDelivered(order);
    default:
      // If you add a new order type and forget to handle it,
      // TypeScript will error here
      const _exhaustive: never = order;
      throw new Error(`Unhandled order type: ${_exhaustive}`);
  }
}
```

### Type Guards

You can create type guard functions for cleaner code:

```typescript
function isShipped(order: Order): order is ShippedOrder {
  return order.kind === 'shipped';
}

function isDelivered(order: Order): order is DeliveredOrder {
  return order.kind === 'delivered';
}

if (isShipped(order)) {
  // TypeScript knows order is ShippedOrder here
  console.log(order.trackingNumber);
}
```

## Advanced Example: Payment Methods

```typescript
type Payment =
  | { readonly kind: 'cash'; readonly amount: number }
  | { readonly kind: 'credit'; readonly amount: number; readonly cardNumber: string; readonly cvv: string }
  | { readonly kind: 'bank'; readonly amount: number; readonly accountNumber: string; readonly routingNumber: string }
  | { readonly kind: 'crypto'; readonly amount: number; readonly walletAddress: string; readonly blockchain: string };

function processFee(payment: Payment): number {
  switch (payment.kind) {
    case 'cash':
      return 0;  // No fee for cash
    
    case 'credit':
      return payment.amount * 0.025;  // 2.5% fee
    
    case 'bank':
      return 1.00;  // Flat $1 fee
    
    case 'crypto':
      return payment.amount * 0.01;  // 1% fee
  }
}

function validatePayment(payment: Payment): boolean {
  switch (payment.kind) {
    case 'cash':
      return payment.amount > 0;
    
    case 'credit':
      return payment.amount > 0 && 
             payment.cardNumber.length === 16 && 
             payment.cvv.length === 3;
    
    case 'bank':
      return payment.amount > 0 && 
             payment.accountNumber.length > 0 && 
             payment.routingNumber.length === 9;
    
    case 'crypto':
      return payment.amount > 0 && 
             payment.walletAddress.length > 0;
  }
}
```

## Benefits

- **Impossible to create invalid states**: Each variant has exactly the fields it needs
- **Type-safe access**: TypeScript knows which fields exist after discriminating
- **Exhaustiveness checking**: Compiler ensures all cases are handled
- **Better refactoring**: Adding a new variant causes compile errors everywhere that needs updating
- **Self-documenting**: The type structure shows all possible states

## See Also

- [Enum to Class Hierarchy](./enum-to-class-hierarchy.md)
- [Flag Arguments](./flag-arguments.md)
- [Honest Functions](./honest-functions.md)
- [Value Semantics](./value-semantics.md)
