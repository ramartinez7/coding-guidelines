# Async Pipe Over Subscribe

> Prefer the async pipe in templates over manual subscriptions to prevent memory leaks and simplify code.

## Problem

Manually subscribing to observables in component code requires careful cleanup in `ngOnDestroy` to prevent memory leaks. It also leads to imperative state management and more boilerplate code.

## Example

### ❌ Before

```typescript
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error">{{ error }}</div>
    <ul>
      <li *ngFor="let user of users">{{ user.name }}</li>
    </ul>
  `
})
export class UserListComponent implements OnInit, OnDestroy {
  users: User[] = [];
  loading = false;
  error: string | null = null;
  private subscription = new Subscription();

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.loading = true;
    
    this.subscription.add(
      this.userService.getUsers().subscribe({
        next: users => {
          this.users = users;
          this.loading = false;
        },
        error: err => {
          this.error = err.message;
          this.loading = false;
        }
      })
    );
  }

  ngOnDestroy() {
    this.subscription.unsubscribe(); // Easy to forget!
  }
}
```

### ✅ After

```typescript
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngIf="users$ | async as users; else loading">
      <ul>
        <li *ngFor="let user of users">{{ user.name }}</li>
      </ul>
    </div>
    <ng-template #loading>
      <div>Loading...</div>
    </ng-template>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserListComponent {
  users$ = this.userService.getUsers().pipe(
    catchError(error => {
      console.error('Failed to load users:', error);
      return of([]); // Return empty array on error
    })
  );

  constructor(private userService: UserService) {}
  
  // No ngOnDestroy needed! Async pipe handles unsubscription
}
```

## Why It's a Problem

1. **Memory leaks**: Forgetting to unsubscribe causes memory leaks and continued processing.

2. **Boilerplate code**: Need to implement `OnDestroy`, create `Subscription` objects, and manage cleanup.

3. **Manual change detection**: Must manually trigger change detection or use default strategy.

4. **Error-prone**: Easy to forget unsubscription, especially with multiple subscriptions.

5. **Testing complexity**: More code to test and mock.

## Symptoms

- Components implementing `OnDestroy` just to unsubscribe
- `Subscription` objects and `.add()` calls throughout the component
- State properties (`users`, `loading`, `error`) that mirror observable values
- `ChangeDetectionStrategy.Default` used because of manual subscriptions
- Memory leaks in long-running applications

## Benefits

- **Automatic cleanup**: Async pipe automatically unsubscribes when component is destroyed
- **Less code**: No need for `OnDestroy`, `Subscription` management, or state properties
- **OnPush strategy**: Works perfectly with `OnPush` change detection for better performance
- **Declarative**: Data flow is clear from template—no hidden subscriptions in component code
- **Composable**: Easy to combine multiple observables with RxJS operators

## Advanced Patterns

### Multiple Async Pipes (Same Observable)

```typescript
// ❌ Multiple subscriptions
@Component({
  template: `
    <div>Count: {{ count$ | async }}</div>
    <div>Doubled: {{ (count$ | async)! * 2 }}</div>
  `
})
export class Component {
  count$ = interval(1000); // Creates TWO subscriptions!
}

// ✅ Share the subscription
@Component({
  template: `
    <div *ngIf="count$ | async as count">
      <div>Count: {{ count }}</div>
      <div>Doubled: {{ count * 2 }}</div>
    </div>
  `
})
export class Component {
  count$ = interval(1000).pipe(shareReplay(1));
}
```

### Error Handling in Template

```typescript
@Component({
  template: `
    <div *ngIf="users$ | async as users; else errorOrLoading">
      <ul><li *ngFor="let user of users">{{ user.name }}</li></ul>
    </div>
    <ng-template #errorOrLoading>
      <div *ngIf="error$ | async as error; else loading">
        Error: {{ error }}
      </div>
      <ng-template #loading>Loading...</ng-template>
    </ng-template>
  `
})
export class UserListComponent {
  users$ = this.userService.getUsers();
  error$ = this.users$.pipe(
    catchError(err => {
      this.errorSubject.next(err.message);
      return EMPTY;
    })
  );
  
  private errorSubject = new Subject<string>();
}
```

### Loading State

```typescript
@Component({
  template: `
    <div *ngIf="!(loading$ | async); else loadingTemplate">
      <div *ngIf="users$ | async as users">
        <ul><li *ngFor="let user of users">{{ user.name }}</li></ul>
      </div>
    </div>
    <ng-template #loadingTemplate>Loading...</ng-template>
  `
})
export class UserListComponent {
  private loadingSubject = new BehaviorSubject<boolean>(true);
  loading$ = this.loadingSubject.asObservable();
  
  users$ = this.userService.getUsers().pipe(
    tap(() => this.loadingSubject.next(false)),
    catchError(err => {
      this.loadingSubject.next(false);
      return throwError(() => err);
    })
  );
}
```

## When Manual Subscription Is Acceptable

There are legitimate cases for manual subscriptions:

1. **Side effects without display**: Logging, analytics, or actions that don't affect the view
2. **Form submissions**: One-time actions where you don't display the result
3. **Imperative navigation**: Redirecting after an action completes

```typescript
// Acceptable: one-time action with side effect
saveUser(user: User) {
  this.userService.save(user).subscribe({
    next: () => this.router.navigate(['/users']),
    error: err => this.errorService.handle(err)
  });
}
```

Even in these cases, consider using operators like `tap()` and the async pipe for consistency.

## Migration Strategy

1. Replace state properties with observables ending in `$`
2. Use `| async` in templates instead of reading state properties
3. Remove `ngOnDestroy` and subscription management code
4. Switch to `OnPush` change detection
5. Use RxJS operators (`catchError`, `map`, `tap`) to handle logic

## See Also

- [Declarative Data Loading](./declarative-data-loading.md)
- [OnPush Change Detection](./onpush-change-detection.md)
- [RxJS Error Handling](./rxjs-error-handling.md)
- [Smart and Presentational Components](./smart-presentational-components.md)
