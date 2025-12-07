# Declarative Data Loading

> Use observables and the async pipe for declarative, reactive data loading instead of imperative state management.

## Problem

Imperatively loading data in lifecycle hooks requires managing loading states, error handling, and cleanup manually. This leads to boilerplate code and potential memory leaks.

## Example

### ❌ Before (Imperative)

```typescript
@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error">Error: {{ error }}</div>
    <div *ngIf="user && !loading && !error">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserProfileComponent implements OnInit, OnDestroy {
  user?: User;
  loading = false;
  error?: string;
  private destroy$ = new Subject<void>();

  constructor(
    private userService: UserService,
    private route: ActivatedRoute
  ) {}

  ngOnInit() {
    this.route.paramMap.pipe(
      takeUntil(this.destroy$)
    ).subscribe(params => {
      const id = params.get('id');
      if (id) {
        this.loadUser(id);
      }
    });
  }

  loadUser(id: string) {
    this.loading = true;
    this.error = undefined;
    
    this.userService.getUser(id).pipe(
      takeUntil(this.destroy$)
    ).subscribe({
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

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### ✅ After (Declarative)

```typescript
@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="user$ | async as user; else loading">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
    <ng-template #loading>
      <div>Loading...</div>
    </ng-template>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserProfileComponent {
  user$ = this.route.paramMap.pipe(
    map(params => params.get('id')!),
    switchMap(id => this.userService.getUser(id)),
    catchError(error => {
      console.error('Failed to load user:', error);
      return of(null);
    })
  );

  constructor(
    private userService: UserService,
    private route: ActivatedRoute
  ) {}
  
  // No ngOnInit, no ngOnDestroy, no manual state management!
}
```

## Why It's a Problem

1. **Boilerplate**: Manual state management for loading, error, and data states.

2. **Memory leaks**: Easy to forget cleanup with `takeUntil` or unsubscribe.

3. **Complex logic**: Nested subscriptions and callbacks.

4. **Race conditions**: Multiple simultaneous requests can cause state inconsistency.

5. **Hard to test**: More code paths and state combinations to test.

## Symptoms

- Components with `loading`, `error`, and data properties
- `ngOnInit` with subscription logic
- `ngOnDestroy` with cleanup code
- Multiple `takeUntil` operators
- Nested subscriptions
- Race condition bugs when switching routes quickly

## Benefits

- **Less code**: No manual state, no lifecycle hooks, no cleanup
- **No memory leaks**: Async pipe handles unsubscription
- **Cleaner logic**: Data flow is clear and declarative
- **Automatic cancellation**: Switching routes cancels previous requests
- **Better performance**: Works with OnPush change detection
- **Easier testing**: Test pure observable streams

## Advanced Patterns

### Loading and Error States

```typescript
interface DataState<T> {
  loading: boolean;
  data: T | null;
  error: string | null;
}

@Component({
  template: `
    <ng-container *ngIf="state$ | async as state">
      <div *ngIf="state.loading">Loading...</div>
      <div *ngIf="state.error">Error: {{ state.error }}</div>
      <div *ngIf="state.data as data">
        <h2>{{ data.name }}</h2>
      </div>
    </ng-container>
  `
})
export class Component {
  state$ = this.route.paramMap.pipe(
    map(params => params.get('id')!),
    switchMap(id => 
      this.service.getData(id).pipe(
        map(data => ({ loading: false, data, error: null })),
        startWith({ loading: true, data: null, error: null }),
        catchError(error => of({ 
          loading: false, 
          data: null, 
          error: error.message 
        }))
      )
    )
  );
}
```

### Combining Multiple Data Sources

```typescript
@Component({
  template: `
    <div *ngIf="viewModel$ | async as vm">
      <h1>{{ vm.user.name }}</h1>
      <ul>
        <li *ngFor="let post of vm.posts">{{ post.title }}</li>
      </ul>
      <div>Comments: {{ vm.commentCount }}</div>
    </div>
  `
})
export class ProfileComponent {
  private userId$ = this.route.paramMap.pipe(
    map(params => params.get('id')!)
  );
  
  private user$ = this.userId$.pipe(
    switchMap(id => this.userService.getUser(id))
  );
  
  private posts$ = this.userId$.pipe(
    switchMap(id => this.postService.getUserPosts(id))
  );
  
  private comments$ = this.userId$.pipe(
    switchMap(id => this.commentService.getUserComments(id))
  );
  
  // Combine into single view model
  viewModel$ = combineLatest([
    this.user$,
    this.posts$,
    this.comments$
  ]).pipe(
    map(([user, posts, comments]) => ({
      user,
      posts,
      commentCount: comments.length
    }))
  );
}
```

### Retry Logic

```typescript
@Component({...})
export class DataComponent {
  data$ = this.http.get<Data>('/api/data').pipe(
    retry({
      count: 3,
      delay: 1000
    }),
    catchError(error => {
      this.errorService.handle(error);
      return of(null);
    })
  );
}
```

### Polling

```typescript
@Component({...})
export class StatusComponent {
  status$ = interval(5000).pipe(
    startWith(0),
    switchMap(() => this.statusService.getStatus()),
    catchError(error => {
      console.error('Status check failed:', error);
      return of({ status: 'unknown' });
    })
  );
}
```

### Dependent Requests

```typescript
@Component({...})
export class OrderComponent {
  order$ = this.route.paramMap.pipe(
    map(params => params.get('orderId')!),
    switchMap(orderId => this.orderService.getOrder(orderId))
  );
  
  // Load customer data based on order
  customer$ = this.order$.pipe(
    switchMap(order => this.customerService.getCustomer(order.customerId))
  );
  
  // Load products for order items
  products$ = this.order$.pipe(
    switchMap(order => {
      const productIds = order.items.map(i => i.productId);
      return forkJoin(
        productIds.map(id => this.productService.getProduct(id))
      );
    })
  );
  
  // Combine everything
  viewModel$ = combineLatest([
    this.order$,
    this.customer$,
    this.products$
  ]).pipe(
    map(([order, customer, products]) => ({
      order,
      customer,
      products
    }))
  );
}
```

### Caching with ShareReplay

```typescript
@Component({...})
export class Component {
  // Cache the result - only one HTTP request
  data$ = this.http.get<Data>('/api/data').pipe(
    shareReplay({ bufferSize: 1, refCount: true })
  );
  
  // Multiple subscriptions use cached value
  filteredData$ = this.data$.pipe(
    map(data => data.items.filter(i => i.active))
  );
  
  totalCount$ = this.data$.pipe(
    map(data => data.items.length)
  );
}
```

### Refresh on Demand

```typescript
@Component({
  template: `
    <div *ngIf="data$ | async as data">
      {{ data | json }}
    </div>
    <button (click)="refresh()">Refresh</button>
  `
})
export class Component {
  private refresh$ = new Subject<void>();
  
  data$ = merge(
    of(undefined), // Initial load
    this.refresh$  // Manual refresh
  ).pipe(
    switchMap(() => this.service.getData())
  );
  
  refresh() {
    this.refresh$.next();
  }
}
```

### Search with Debounce

```typescript
@Component({
  template: `
    <input [formControl]="searchControl">
    <div *ngIf="results$ | async as results">
      <div *ngFor="let result of results">{{ result.name }}</div>
    </div>
  `
})
export class SearchComponent {
  searchControl = new FormControl('');
  
  results$ = this.searchControl.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    filter(term => term.length >= 3),
    switchMap(term => this.searchService.search(term)),
    catchError(error => {
      console.error('Search failed:', error);
      return of([]);
    })
  );
}
```

### Optimistic Updates

```typescript
@Component({...})
export class TodoComponent {
  private todosSubject = new BehaviorSubject<Todo[]>([]);
  todos$ = this.todosSubject.asObservable();
  
  constructor(private todoService: TodoService) {
    this.loadTodos();
  }
  
  private loadTodos() {
    this.todoService.getTodos().subscribe(
      todos => this.todosSubject.next(todos)
    );
  }
  
  addTodo(text: string) {
    const optimisticTodo = { id: Date.now(), text, completed: false };
    
    // Update UI immediately (optimistic)
    this.todosSubject.next([...this.todosSubject.value, optimisticTodo]);
    
    // Send to server
    this.todoService.createTodo(text).pipe(
      catchError(error => {
        // Revert on error
        this.todosSubject.next(
          this.todosSubject.value.filter(t => t.id !== optimisticTodo.id)
        );
        return throwError(() => error);
      })
    ).subscribe(serverTodo => {
      // Replace optimistic with server version
      this.todosSubject.next(
        this.todosSubject.value.map(t => 
          t.id === optimisticTodo.id ? serverTodo : t
        )
      );
    });
  }
}
```

## Handling Edge Cases

### Empty States

```typescript
@Component({
  template: `
    <div *ngIf="items$ | async as items; else loading">
      <div *ngIf="items.length === 0; else list">
        No items found.
      </div>
      <ng-template #list>
        <div *ngFor="let item of items">{{ item.name }}</div>
      </ng-template>
    </div>
    <ng-template #loading>Loading...</ng-template>
  `
})
```

### Null vs Loading

```typescript
// Use BehaviorSubject with undefined for loading state
private dataSubject = new BehaviorSubject<Data | undefined>(undefined);
data$ = this.dataSubject.asObservable();

@Component({
  template: `
    <ng-container *ngIf="data$ | async as data; else loading">
      <div>{{ data.value }}</div>
    </ng-container>
    <ng-template #loading>
      <div *ngIf="(data$ | async) === undefined; else error">
        Loading...
      </div>
      <ng-template #error>Error loading data</ng-template>
    </ng-template>
  `
})
```

## Testing Declarative Components

```typescript
describe('UserProfileComponent', () => {
  it('should load user data', fakeAsync(() => {
    const mockUser = { id: '1', name: 'Test User' };
    const userService = jasmine.createSpyObj('UserService', ['getUser']);
    userService.getUser.and.returnValue(of(mockUser));
    
    const component = new UserProfileComponent(userService, mockRoute);
    
    component.user$.subscribe(user => {
      expect(user).toEqual(mockUser);
    });
    
    tick();
  }));
});
```

## See Also

- [Async Pipe Over Subscribe](./async-pipe-over-subscribe.md)
- [RxJS Error Handling](./rxjs-error-handling.md)
- [Smart and Presentational Components](./smart-presentational-components.md)
- [OnPush Change Detection](./onpush-change-detection.md)
