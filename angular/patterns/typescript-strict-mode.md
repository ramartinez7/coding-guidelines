# TypeScript Strict Mode

> Enable TypeScript's strict mode for maximum type safety and early error detection.

## Problem

Without strict mode, TypeScript allows many unsafe patterns that can lead to runtime errors. These issues pass through the type checker but fail in production.

## Example

### ❌ Before (tsconfig.json)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "lib": ["ES2022", "dom"],
    "strict": false
  }
}
```

```typescript
// This code compiles but is full of potential bugs
export class UserService {
  private users; // Implicitly 'any'
  
  getUser(id) { // Parameters implicitly 'any'
    return this.users.find(u => u.id === id); // Might return undefined
  }
  
  processUser(user: User) {
    console.log(user.name.toUpperCase()); // What if name is null?
  }
}
```

### ✅ After (tsconfig.json)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "lib": ["ES2022", "dom"],
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

```typescript
// Explicit types catch errors at compile time
export class UserService {
  private users: User[] = [];
  
  getUser(id: number): User | undefined {
    return this.users.find(u => u.id === id);
  }
  
  processUser(user: User): void {
    // TypeScript forces you to handle null/undefined
    if (user.name) {
      console.log(user.name.toUpperCase());
    }
  }
}
```

## Why It's a Problem

1. **Runtime errors**: Type issues slip through compilation and fail in production.

2. **Implicit any**: Variables and parameters without types become `any`, losing all type safety.

3. **Null pointer exceptions**: `null` and `undefined` are treated as valid values for any type.

4. **False confidence**: Code appears type-safe but isn't actually checked properly.

5. **Refactoring risks**: Changes break code in ways the compiler doesn't catch.

## Symptoms

- Frequent `TypeError` or `Cannot read property of undefined` in production
- Variables or parameters without explicit types
- No compiler errors despite obvious type mismatches
- `any` type appearing throughout the codebase
- Defensive `if (x != null)` checks everywhere

## Benefits

- **Catch errors early**: Find bugs at compile time instead of runtime
- **Better IntelliSense**: More accurate autocomplete and documentation
- **Safer refactoring**: Compiler catches breaking changes
- **Self-documenting**: Types serve as inline documentation
- **Fewer defensive checks**: Type system proves safety

## Key Strict Mode Flags

### `noImplicitAny`

Forces explicit types on all declarations.

```typescript
// ❌ Without noImplicitAny
function add(a, b) { // Implicitly 'any'
  return a + b;
}

// ✅ With noImplicitAny
function add(a: number, b: number): number {
  return a + b;
}
```

### `strictNullChecks`

Makes `null` and `undefined` separate types that must be explicitly handled.

```typescript
// ❌ Without strictNullChecks
function getLength(str: string): number {
  return str.length; // str could be null!
}

// ✅ With strictNullChecks
function getLength(str: string | null): number {
  if (str === null) {
    return 0;
  }
  return str.length; // TypeScript knows str is not null here
}

// Or use optional chaining
function getLength(str: string | null): number {
  return str?.length ?? 0;
}
```

### `strictPropertyInitialization`

Ensures class properties are initialized.

```typescript
// ❌ Without strictPropertyInitialization
export class UserComponent {
  user: User; // Not initialized! Runtime error waiting to happen
  
  ngOnInit() {
    console.log(this.user.name); // Error: user is undefined
  }
}

// ✅ With strictPropertyInitialization
export class UserComponent {
  // Option 1: Initialize in declaration
  user: User = { id: 0, name: '', email: '' };
  
  // Option 2: Use definite assignment assertion (when you know it's set elsewhere)
  @Input() user!: User;
  
  // Option 3: Make it optional
  user?: User;
}
```

### `noImplicitReturns`

Ensures all code paths in a function return a value.

```typescript
// ❌ Without noImplicitReturns
function getStatus(code: number): string {
  if (code === 200) {
    return 'OK';
  }
  if (code === 404) {
    return 'Not Found';
  }
  // Missing return for other codes! TypeScript allows this.
}

// ✅ With noImplicitReturns
function getStatus(code: number): string {
  if (code === 200) {
    return 'OK';
  }
  if (code === 404) {
    return 'Not Found';
  }
  return 'Unknown'; // Compiler enforces this
}
```

## Angular-Specific Configuration

For Angular projects, also enable Angular's strict mode:

```json
{
  "angularCompilerOptions": {
    "strictTemplates": true,
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictAttributeTypes": true,
    "strictSafeNavigationTypes": true,
    "strictDomLocalRefTypes": true,
    "strictOutputEventTypes": true,
    "strictDomEventTypes": true
  }
}
```

This enables type checking in templates:

```typescript
@Component({
  template: `
    <div>{{ user.name }}</div>  <!-- TypeScript checks this! -->
    <button (click)="save(user)">Save</button>  <!-- Checks method signature -->
  `
})
export class UserComponent {
  user?: User; // Template checks will catch missing null check
  
  save(user: User): void {
    // ...
  }
}
```

## Migration Strategy

Enabling strict mode in an existing project requires gradual migration:

1. **Enable one flag at a time**:
   ```json
   {
     "compilerOptions": {
       "strict": false,
       "noImplicitAny": true  // Start here
     }
   }
   ```

2. **Fix errors incrementally**: Address one file or module at a time.

3. **Use `any` as escape hatch temporarily**:
   ```typescript
   // TODO: Add proper types
   const data: any = JSON.parse(response);
   ```

4. **Enable more flags**: Once `noImplicitAny` is clean, add `strictNullChecks`.

5. **Eventually enable all**: Set `"strict": true` when ready.

## Common Patterns with Strict Mode

### Handling Nullable Values

```typescript
// Use optional chaining
const name = user?.profile?.name;

// Use nullish coalescing
const displayName = user?.name ?? 'Anonymous';

// Type narrowing with type guards
if (user !== null && user !== undefined) {
  console.log(user.name); // TypeScript knows user is defined
}
```

### Definite Assignment Assertion

```typescript
export class Component {
  @Input() userId!: string; // '!' tells TypeScript it WILL be set
  
  ngOnInit() {
    // userId is guaranteed to be set because Angular lifecycle
    this.loadUser(this.userId);
  }
}
```

Use `!` sparingly and only when you're certain the value will be set.

## See Also

- [Template Type Checking](./template-type-checking.md)
- [Type-Safe Forms](./type-safe-forms.md)
- [Service Abstractions](./service-abstractions.md)
