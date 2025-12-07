# Philosophy

> These patterns sit at the intersection of Angular Best Practices, Reactive Programming, and Type Safety.

This document explains the *why* behind our Angular coding patterns. Each pattern in our catalog draws from one or more of these schools of thought.

---

## 1. Component-Driven Architecture

Angular applications are built from components. This philosophy argues that **components should be small, focused, and composable**.

### Core Concepts

**Smart vs. Presentational Components**  
Separating components that manage state and logic from those that only present data.

```typescript
// Presentational: only displays data
@Component({
  selector: 'app-user-card',
  template: `<div>{{ user.name }}</div>`
})
export class UserCardComponent {
  @Input() user!: User;
}

// Smart: manages state and logic
@Component({
  selector: 'app-user-list',
  template: `<app-user-card *ngFor="let user of users$ | async" [user]="user" />`
})
export class UserListComponent {
  users$ = this.userService.getUsers();
  constructor(private userService: UserService) {}
}
```

**Single Responsibility**  
Each component should have one clear purpose. Components that do too much are difficult to test, reuse, and maintain.

**Composition Over Inheritance**  
Favor component composition and Angular's dependency injection over class inheritance.

### Related Patterns

- [Smart and Presentational Components](./patterns/smart-presentational-components.md)
- [Component Composition](./patterns/component-composition.md)
- [Input/Output Best Practices](./patterns/input-output-best-practices.md)

---

## 2. Reactive Programming with RxJS

Angular embraces reactive programming through RxJS. This philosophy promotes **declarative data flows** over imperative state management.

### The Golden Rule

**Think in Streams, Not Values.**

Instead of managing state imperatively, declare how data flows through your application.

```typescript
// ❌ Imperative: manual state management
export class UserComponent {
  users: User[] = [];
  loading = false;
  error: string | null = null;

  loadUsers() {
    this.loading = true;
    this.userService.getUsers().subscribe({
      next: users => {
        this.users = users;
        this.loading = false;
      },
      error: err => {
        this.error = err.message;
        this.loading = false;
      }
    });
  }
}

// ✅ Reactive: declarative data flow
export class UserComponent {
  users$ = this.userService.getUsers().pipe(
    catchError(error => {
      this.errorService.handle(error);
      return EMPTY;
    })
  );
  
  constructor(
    private userService: UserService,
    private errorService: ErrorService
  ) {}
}
```

### Avoid Manual Subscriptions

Use the `async` pipe in templates to handle subscriptions automatically—prevents memory leaks and simplifies code.

### Related Patterns

- [Async Pipe Over Subscribe](./patterns/async-pipe-over-subscribe.md)
- [Declarative Data Loading](./patterns/declarative-data-loading.md)
- [RxJS Error Handling](./patterns/rxjs-error-handling.md)

---

## 3. Type Safety with TypeScript

Angular is built with TypeScript. This philosophy emphasizes **leveraging the type system** to catch errors at compile time.

### Core Principles

**Strong Typing Everywhere**  
Don't use `any` or loose types. Every value should have a precise type.

```typescript
// ❌ Weak typing
function processData(data: any) {
  return data.items.map((x: any) => x.value);
}

// ✅ Strong typing
interface DataResponse {
  items: Array<{ value: number }>;
}

function processData(data: DataResponse): number[] {
  return data.items.map(item => item.value);
}
```

**Strict Mode**  
Enable TypeScript's strict mode and Angular's strict template checking to maximize type safety.

### Related Patterns

- [TypeScript Strict Mode](./patterns/typescript-strict-mode.md)
- [Type-Safe Forms](./patterns/type-safe-forms.md)
- [Template Type Checking](./patterns/template-type-checking.md)

---

## 4. Dependency Injection and Testability

Angular's dependency injection system enables **loose coupling and testability**. This philosophy promotes designing for injectability.

### Core Concepts

**Inject Services, Not Concretions**  
Depend on abstractions (interfaces) rather than concrete implementations.

```typescript
// ❌ Direct dependency on concrete class
export class UserComponent {
  private http = new HttpClient(/*...*/);
}

// ✅ Dependency injected
export class UserComponent {
  constructor(private userService: UserService) {}
}
```

**Constructor Injection**  
All dependencies should be declared in the constructor for clarity and testability.

### Related Patterns

- [Service Abstractions](./patterns/service-abstractions.md)
- [Testing with Mock Services](./patterns/testing-mock-services.md)
- [Injection Tokens](./patterns/injection-tokens.md)

---

## 5. Unidirectional Data Flow

Data should flow in one direction: from parent to child through `@Input()`, and from child to parent through `@Output()`.

### The Rule

**Never Mutate Input Properties.**

Input properties should be treated as read-only. Changes should flow through outputs.

```typescript
// ❌ Mutating input
@Component({...})
export class ChildComponent {
  @Input() user!: User;
  
  updateName(name: string) {
    this.user.name = name; // DON'T mutate input!
  }
}

// ✅ Emit changes via output
@Component({...})
export class ChildComponent {
  @Input() user!: User;
  @Output() userChange = new EventEmitter<User>();
  
  updateName(name: string) {
    this.userChange.emit({ ...this.user, name });
  }
}
```

### Related Patterns

- [Input/Output Best Practices](./patterns/input-output-best-practices.md)
- [OnPush Change Detection](./patterns/onpush-change-detection.md)
- [Immutable Data Patterns](./patterns/immutable-data-patterns.md)

---

## 6. Performance and Change Detection

Angular's change detection can be optimized through **strategic use of `OnPush` strategy** and immutable data patterns.

### Core Principles

**OnPush by Default**  
Use `OnPush` change detection strategy for better performance. This requires immutable data patterns.

**Trackby Functions**  
Always provide `trackBy` functions for `*ngFor` to optimize list rendering.

```typescript
// ❌ No trackBy
<div *ngFor="let user of users">{{ user.name }}</div>

// ✅ With trackBy
<div *ngFor="let user of users; trackBy: trackById">{{ user.name }}</div>

trackById(index: number, user: User): number {
  return user.id;
}
```

### Related Patterns

- [OnPush Change Detection](./patterns/onpush-change-detection.md)
- [TrackBy Functions](./patterns/trackby-functions.md)
- [Immutable Data Patterns](./patterns/immutable-data-patterns.md)

---

## 7. Module Organization and Lazy Loading

Structure applications with **feature modules** and lazy load routes for optimal bundle size.

### Core Concepts

**Feature Modules**  
Group related components, services, and routes into feature modules.

**Lazy Loading**  
Load feature modules only when needed to reduce initial bundle size.

```typescript
// Lazy-loaded route
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  }
];
```

**Core and Shared Modules**  
- Core Module: singleton services loaded once
- Shared Module: reusable components, directives, and pipes

### Related Patterns

- [Feature Module Structure](./patterns/feature-module-structure.md)
- [Lazy Loading Strategies](./patterns/lazy-loading-strategies.md)
- [Core vs Shared Modules](./patterns/core-shared-modules.md)

---

## 8. Signals and Modern Angular (v16+)

Angular's new Signals API provides a **simpler, more efficient reactivity model** compared to RxJS for component state.

### When to Use Signals vs RxJS

**Use Signals for:**
- Component-local state
- Derived computed values
- Simple reactive updates

**Use RxJS for:**
- Asynchronous operations (HTTP, timers)
- Complex event streams
- Multi-source combinations

```typescript
// Signals for component state
export class CounterComponent {
  count = signal(0);
  doubled = computed(() => this.count() * 2);
  
  increment() {
    this.count.update(c => c + 1);
  }
}

// RxJS for async operations
export class UserComponent {
  users$ = this.http.get<User[]>('/api/users').pipe(
    retry(3),
    catchError(this.handleError)
  );
}
```

### Related Patterns

- [Signals for Component State](./patterns/signals-component-state.md)
- [Signals vs RxJS](./patterns/signals-vs-rxjs.md)
- [Computed Values](./patterns/computed-values.md)

---

## Summary

These eight philosophies reinforce each other:

| Philosophy | Key Question |
|------------|--------------|
| **Component-Driven** | Is this component focused and composable? |
| **Reactive Programming** | Am I thinking in streams or managing state manually? |
| **Type Safety** | Can the compiler catch this error? |
| **Dependency Injection** | Are my components testable and loosely coupled? |
| **Unidirectional Data Flow** | Does data flow clearly from parent to child? |
| **Performance** | Am I using OnPush and trackBy appropriately? |
| **Module Organization** | Are features properly encapsulated and lazy-loaded? |
| **Modern Angular** | Should this be a Signal or an Observable? |

When reviewing code or designing new features, ask these questions. The patterns in this catalog are tools to answer "yes" to each one.
