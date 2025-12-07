# Smart and Presentational Components

> Separate components that manage state and logic from those that only display data.

## Problem

Components that mix data fetching, business logic, and presentation become difficult to test, reuse, and maintain. They violate the single responsibility principle and create tight coupling.

## Example

### ❌ Before

```typescript
@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error">{{ error }}</div>
    <div *ngIf="user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
      <button (click)="updateUser()">Update</button>
    </div>
  `
})
export class UserProfileComponent implements OnInit {
  user?: User;
  loading = false;
  error?: string;

  constructor(
    private userService: UserService,
    private route: ActivatedRoute,
    private router: Router
  ) {}

  ngOnInit() {
    this.loading = true;
    const id = this.route.snapshot.paramMap.get('id')!;
    
    this.userService.getUser(id).subscribe({
      next: user => {
        this.user = user;
        this.loading = false;
      },
      error: err => {
        this.error = err.message;
        this.loading = false;
      }
    });
  }

  updateUser() {
    if (!this.user) return;
    
    this.userService.updateUser(this.user).subscribe({
      next: () => this.router.navigate(['/users']),
      error: err => this.error = err.message
    });
  }
}
```

### ✅ After

```typescript
// Presentational Component: Only displays data
@Component({
  selector: 'app-user-card',
  template: `
    <div class="user-card">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
      <button (click)="update.emit(user)">Update</button>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserCardComponent {
  @Input() user!: User;
  @Output() update = new EventEmitter<User>();
}

// Smart Component: Manages state and logic
@Component({
  selector: 'app-user-profile-page',
  template: `
    <div *ngIf="loading$ | async">Loading...</div>
    <div *ngIf="error$ | async as error">{{ error }}</div>
    <app-user-card 
      *ngIf="user$ | async as user"
      [user]="user"
      (update)="onUpdateUser($event)">
    </app-user-card>
  `
})
export class UserProfilePageComponent {
  user$ = this.route.paramMap.pipe(
    switchMap(params => this.userService.getUser(params.get('id')!))
  );
  
  loading$ = this.user$.pipe(
    map(() => false),
    startWith(true)
  );
  
  error$ = this.user$.pipe(
    catchError(err => of(err.message))
  );

  constructor(
    private userService: UserService,
    private route: ActivatedRoute,
    private router: Router
  ) {}

  onUpdateUser(user: User) {
    this.userService.updateUser(user).subscribe({
      next: () => this.router.navigate(['/users']),
      error: err => console.error(err)
    });
  }
}
```

## Why It's a Problem

1. **Difficult to test**: Testing requires mocking multiple services and understanding complex state management.

2. **Hard to reuse**: The presentation logic is tightly coupled to data fetching and routing.

3. **Poor performance**: Can't use OnPush change detection effectively when components manage their own state.

4. **Unclear responsibilities**: Component does too many things—violates Single Responsibility Principle.

## Symptoms

- Components with both `HttpClient` injections and template markup
- Components that directly inject `ActivatedRoute` or `Router` alongside presentation logic
- Difficulty writing unit tests without extensive mocking
- Components that can't be reused in different contexts
- `ChangeDetectionStrategy.Default` required because of manual state management

## Benefits

- **Easier testing**: Presentational components only need input props; smart components test business logic
- **Better reusability**: Presentational components work in any context
- **Improved performance**: Presentational components can use OnPush strategy
- **Clear separation**: Each component has one clear responsibility
- **Better composition**: Mix and match presentational components in different smart components

## Guidelines

### Presentational Components Should:

- Receive data via `@Input()`
- Emit events via `@Output()`
- Use `OnPush` change detection
- Have no dependencies on services (except presentation utilities)
- Be highly reusable across the application
- Focus solely on how things look

### Smart Components Should:

- Manage state and orchestrate data flow
- Inject services for data fetching and business logic
- Handle routing and navigation
- Coordinate multiple presentational components
- Use the `async` pipe for reactive data
- Focus on how things work

## When to Break the Rule

Sometimes a component is truly standalone and won't be reused. For simple CRUD pages or one-off views, combining smart and presentational logic is acceptable if:

- The component is not complex
- It won't be reused elsewhere
- Performance isn't a concern
- The team agrees it improves code clarity

## See Also

- [OnPush Change Detection](./onpush-change-detection.md)
- [Async Pipe Over Subscribe](./async-pipe-over-subscribe.md)
- [Component Composition](./component-composition.md)
- [Declarative Data Loading](./declarative-data-loading.md)
