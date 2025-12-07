# OnPush Change Detection

> Use OnPush strategy to optimize change detection and improve performance by only checking when inputs change or events fire.

## Problem

Angular's default change detection strategy checks every component on every browser event, timer, or async operation. This can cause performance issues in large applications, especially with deep component trees.

## Example

### ❌ Before

```typescript
@Component({
  selector: 'app-user-card',
  template: `
    <div class="card">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
      <p>Last updated: {{ getFormattedTime() }}</p>
    </div>
  `
  // Using default change detection strategy
})
export class UserCardComponent {
  @Input() user!: User;

  getFormattedTime(): string {
    console.log('getFormattedTime called'); // Called on EVERY change detection!
    return new Date().toLocaleTimeString();
  }
}
```

### ✅ After

```typescript
@Component({
  selector: 'app-user-card',
  template: `
    <div class="card">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
      <p>Last updated: {{ formattedTime }}</p>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserCardComponent implements OnInit {
  @Input() user!: User;
  formattedTime = '';

  ngOnInit() {
    this.formattedTime = new Date().toLocaleTimeString();
  }
}

// Or with computed signal (Angular 16+)
@Component({
  selector: 'app-user-card',
  template: `
    <div class="card">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
      <p>Last updated: {{ formattedTime() }}</p>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserCardComponent {
  user = input.required<User>();
  formattedTime = computed(() => this.user().updatedAt.toLocaleTimeString());
}
```

## Why It's a Problem

1. **Performance degradation**: Default strategy checks every component on every event, even if nothing changed.

2. **Wasted CPU cycles**: Complex template expressions re-evaluate constantly.

3. **Scalability issues**: Performance problems compound as the application grows.

4. **Unpredictable behavior**: Component updates at unexpected times, making debugging harder.

## Symptoms

- Slow UI interactions in large component trees
- Methods in templates called repeatedly without reason
- Console logs in component methods firing constantly
- Performance profiling shows excessive change detection cycles
- `ngDoCheck` or similar lifecycle hooks called excessively

## Benefits

- **Improved performance**: Components only check when inputs change or events fire
- **Predictable updates**: Clear understanding of when component updates
- **Better scalability**: Performance stays consistent as app grows
- **Forces good practices**: Encourages immutable data patterns and reactive programming
- **Battery life**: Reduced CPU usage benefits mobile devices

## How OnPush Works

Change detection runs when:

1. **Input reference changes**: `@Input()` property receives a new reference
2. **Event fires**: User interaction within component (click, input, etc.)
3. **Observable emits**: When using `async` pipe
4. **Manual trigger**: Calling `ChangeDetectorRef.markForCheck()`

```typescript
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users">
      <app-user-card [user]="user"></app-user-card>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserListComponent {
  @Input() users!: User[];
  
  // Change detection runs when users array reference changes
  updateUser(index: number, newUser: User) {
    // ❌ Mutating array - OnPush won't detect this!
    this.users[index] = newUser;
    
    // ✅ Creating new array - OnPush detects the reference change
    this.users = [...this.users.slice(0, index), newUser, ...this.users.slice(index + 1)];
    
    // Or more simply:
    this.users = this.users.map((u, i) => i === index ? newUser : u);
  }
}
```

## Immutable Data Pattern

OnPush works best with immutable data:

```typescript
// ❌ Mutable updates don't trigger OnPush
updateUserName(name: string) {
  this.user.name = name; // OnPush won't detect this
}

// ✅ Immutable updates trigger OnPush
updateUserName(name: string) {
  this.user = { ...this.user, name }; // New object reference
}

// ✅ For arrays
addItem(item: Item) {
  this.items = [...this.items, item];
}

removeItem(index: number) {
  this.items = this.items.filter((_, i) => i !== index);
}
```

## Working with Async Operations

OnPush works perfectly with the `async` pipe:

```typescript
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngIf="users$ | async as users">
      <app-user-card *ngFor="let user of users" [user]="user"></app-user-card>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserListComponent {
  users$ = this.userService.getUsers(); // Observable
  
  constructor(private userService: UserService) {}
  
  // async pipe automatically triggers change detection when observable emits
}
```

## Manual Change Detection

Sometimes you need to manually trigger updates:

```typescript
@Component({
  selector: 'app-timer',
  template: `<div>{{ elapsedTime }}s</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class TimerComponent implements OnInit, OnDestroy {
  elapsedTime = 0;
  private intervalId?: number;

  constructor(private cdr: ChangeDetectorRef) {}

  ngOnInit() {
    this.intervalId = window.setInterval(() => {
      this.elapsedTime++;
      this.cdr.markForCheck(); // Manually trigger change detection
    }, 1000);
  }

  ngOnDestroy() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
    }
  }
}

// Better: use RxJS and async pipe instead
@Component({
  selector: 'app-timer',
  template: `<div>{{ elapsedTime$ | async }}s</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class TimerComponent {
  elapsedTime$ = interval(1000).pipe(
    map(count => count + 1)
  );
}
```

## Common Pitfalls

### Mutating Input Objects

```typescript
// Parent component
@Component({
  template: `<app-user-card [user]="user"></app-user-card>`
})
export class ParentComponent {
  user = { name: 'John', email: 'john@example.com' };
  
  updateName(name: string) {
    // ❌ Child with OnPush won't see this change
    this.user.name = name;
    
    // ✅ Child will see this change
    this.user = { ...this.user, name };
  }
}
```

### Shared Mutable Objects

```typescript
// ❌ Sharing mutable state between components
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ComponentA {
  @Input() config!: AppConfig;
  
  updateConfig() {
    this.config.theme = 'dark'; // Modifies shared object
  }
}

// ✅ Emit changes to parent for immutable updates
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ComponentA {
  @Input() config!: AppConfig;
  @Output() configChange = new EventEmitter<AppConfig>();
  
  updateConfig() {
    this.configChange.emit({ ...this.config, theme: 'dark' });
  }
}
```

## When to Use Default Strategy

Rare cases where default strategy is acceptable:

1. **Third-party integrations**: Libraries that mutate DOM directly
2. **Performance non-critical**: Small apps or components used once
3. **Legacy code**: Gradual migration from existing codebase

Even in these cases, OnPush is usually better with proper patterns.

## Migration Checklist

- [ ] Add `changeDetection: ChangeDetectionStrategy.OnPush` to component
- [ ] Replace method calls in templates with properties or pipes
- [ ] Ensure all input mutations create new references
- [ ] Use `async` pipe for observables
- [ ] Use signals for component state (Angular 16+)
- [ ] Update parent components to pass immutable data
- [ ] Test all update scenarios

## See Also

- [Immutable Data Patterns](./immutable-data-patterns.md)
- [Async Pipe Over Subscribe](./async-pipe-over-subscribe.md)
- [Smart and Presentational Components](./smart-presentational-components.md)
- [Signals for Component State](./signals-component-state.md)
