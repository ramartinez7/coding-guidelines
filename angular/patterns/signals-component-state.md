# Signals for Component State

> Use Angular Signals (v16+) for simpler, more efficient component-local state management.

## Problem

Managing component state with RxJS requires boilerplate (`BehaviorSubject`, subscriptions, change detection) for simple cases. Signals provide a more lightweight, intuitive API for reactive component state.

## Example

### ❌ Before (RxJS for Simple State)

```typescript
@Component({
  selector: 'app-counter',
  template: `
    <div>Count: {{ count$ | async }}</div>
    <div>Doubled: {{ doubled$ | async }}</div>
    <button (click)="increment()">+1</button>
    <button (click)="decrement()">-1</button>
    <button (click)="reset()">Reset</button>
  `
})
export class CounterComponent {
  private countState$ = new BehaviorSubject<number>(0);
  count$ = this.countState$.asObservable();
  
  doubled$ = this.count$.pipe(
    map(count => count * 2)
  );

  increment() {
    this.countState$.next(this.countState$.value + 1);
  }

  decrement() {
    this.countState$.next(this.countState$.value - 1);
  }

  reset() {
    this.countState$.next(0);
  }
}
```

### ✅ After (Signals)

```typescript
@Component({
  selector: 'app-counter',
  template: `
    <div>Count: {{ count() }}</div>
    <div>Doubled: {{ doubled() }}</div>
    <button (click)="increment()">+1</button>
    <button (click)="decrement()">-1</button>
    <button (click)="reset()">Reset</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class CounterComponent {
  count = signal(0);
  doubled = computed(() => this.count() * 2);

  increment() {
    this.count.update(c => c + 1);
  }

  decrement() {
    this.count.update(c => c - 1);
  }

  reset() {
    this.count.set(0);
  }
}
```

## Why It's a Problem

1. **Boilerplate**: BehaviorSubject requires more code for simple state.

2. **Cognitive overhead**: Understanding RxJS concepts (subjects, operators, subscriptions) for simple cases.

3. **Memory management**: Need to manage subscriptions or use async pipe.

4. **Complexity**: Derived state requires operators and pipes.

5. **Performance**: Every update triggers change detection even with OnPush.

## Symptoms

- Using `BehaviorSubject` for component-local state
- Computed values requiring `combineLatest` or multiple pipes
- Many `async` pipes in template for simple state
- Complex setup for simple reactive updates

## Benefits

- **Less boilerplate**: Simpler API, less code
- **Better performance**: Fine-grained reactivity updates only what changed
- **Easier to understand**: More intuitive than RxJS for simple cases
- **Type-safe**: Full TypeScript support
- **No subscriptions**: No memory leak concerns
- **Computed values**: Automatic dependency tracking

## Core Signal APIs

### Creating Signals

```typescript
// Writable signal with initial value
const count = signal(0);

// Type inference
const name = signal('Alice'); // WritableSignal<string>
const items = signal<Item[]>([]); // Explicit type

// Signal of complex type
interface User {
  id: number;
  name: string;
}

const user = signal<User>({ id: 1, name: 'Alice' });
```

### Reading Signals

```typescript
// Call as function to read value
const currentCount = count(); // number

// In template
@Component({
  template: `<div>Count: {{ count() }}</div>`
})
```

### Updating Signals

```typescript
// Set absolute value
count.set(10);

// Update based on current value
count.update(current => current + 1);

// For objects/arrays, create new reference
const user = signal({ id: 1, name: 'Alice' });
user.update(u => ({ ...u, name: 'Bob' })); // Immutable update
```

### Computed Signals

```typescript
const count = signal(0);
const doubled = computed(() => this.count() * 2);

// Computed signals automatically track dependencies
const total = computed(() => {
  const a = signalA();
  const b = signalB();
  return a + b; // Recalculates only when a or b change
});

// Lazy evaluation - computed only when read
const expensive = computed(() => {
  return heavyCalculation(data());
}); // Not calculated until expensive() is called
```

### Effects

```typescript
constructor() {
  // Run side effects when signals change
  effect(() => {
    console.log('Count changed:', this.count());
  });
}

// Effects run in injection context
@Component({...})
export class MyComponent {
  count = signal(0);
  
  constructor() {
    effect(() => {
      // Automatically tracks this.count()
      localStorage.setItem('count', this.count().toString());
    });
  }
}
```

## Common Patterns

### Object State

```typescript
interface TodoState {
  items: Todo[];
  filter: 'all' | 'active' | 'completed';
}

@Component({...})
export class TodoComponent {
  state = signal<TodoState>({
    items: [],
    filter: 'all'
  });
  
  // Computed filtered list
  filteredItems = computed(() => {
    const { items, filter } = this.state();
    
    if (filter === 'all') return items;
    if (filter === 'active') return items.filter(t => !t.completed);
    return items.filter(t => t.completed);
  });
  
  addTodo(text: string) {
    this.state.update(s => ({
      ...s,
      items: [...s.items, { id: Date.now(), text, completed: false }]
    }));
  }
  
  setFilter(filter: TodoState['filter']) {
    this.state.update(s => ({ ...s, filter }));
  }
}
```

### Array Operations

```typescript
@Component({...})
export class ItemsComponent {
  items = signal<Item[]>([]);
  
  addItem(item: Item) {
    this.items.update(items => [...items, item]);
  }
  
  removeItem(id: number) {
    this.items.update(items => items.filter(i => i.id !== id));
  }
  
  updateItem(id: number, changes: Partial<Item>) {
    this.items.update(items =>
      items.map(i => i.id === id ? { ...i, ...changes } : i)
    );
  }
  
  clearItems() {
    this.items.set([]);
  }
}
```

### Loading States

```typescript
@Component({...})
export class DataComponent {
  data = signal<Data | null>(null);
  loading = signal(false);
  error = signal<string | null>(null);
  
  // Computed derived state
  hasData = computed(() => this.data() !== null);
  isEmpty = computed(() => !this.loading() && !this.hasData());
  
  async loadData() {
    this.loading.set(true);
    this.error.set(null);
    
    try {
      const result = await this.api.getData();
      this.data.set(result);
    } catch (err) {
      this.error.set(err.message);
    } finally {
      this.loading.set(false);
    }
  }
}
```

### Form State

```typescript
@Component({...})
export class FormComponent {
  formData = signal({
    name: '',
    email: '',
    age: 0
  });
  
  // Validation as computed
  isValid = computed(() => {
    const data = this.formData();
    return data.name.length > 0 && 
           data.email.includes('@') && 
           data.age >= 18;
  });
  
  updateField<K extends keyof typeof this.formData.value>(
    field: K,
    value: typeof this.formData.value[K]
  ) {
    this.formData.update(form => ({ ...form, [field]: value }));
  }
  
  submit() {
    if (this.isValid()) {
      console.log('Submitting:', this.formData());
    }
  }
}
```

## Signals vs RxJS

### Use Signals for:

- Component-local state
- Synchronous derived values
- Simple reactive updates
- UI state (toggles, counters, form data)

```typescript
// ✅ Good use of Signals
count = signal(0);
isEven = computed(() => this.count() % 2 === 0);
```

### Use RxJS for:

- Asynchronous operations (HTTP, timers, WebSocket)
- Complex event streams
- Combining multiple async sources
- Time-based operations (debounce, throttle)

```typescript
// ✅ Good use of RxJS
users$ = this.http.get<User[]>('/api/users').pipe(
  retry(3),
  catchError(this.handleError)
);

search$ = this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.search(term))
);
```

### Combining Both

```typescript
@Component({...})
export class SearchComponent {
  // Signal for UI state
  selectedFilter = signal<'all' | 'active'>('all');
  
  // RxJS for async operations
  results$ = this.searchControl.valueChanges.pipe(
    debounceTime(300),
    switchMap(term => this.searchService.search(term))
  );
  
  // Convert Observable to Signal (Angular 16.2+)
  results = toSignal(this.results$, { initialValue: [] });
  
  // Computed combining signal and converted observable
  filteredResults = computed(() => {
    const filter = this.selectedFilter();
    const items = this.results();
    
    return filter === 'all' 
      ? items 
      : items.filter(i => i.status === 'active');
  });
}
```

## Signal Inputs (Angular 17.1+)

```typescript
@Component({...})
export class UserCardComponent {
  // Traditional input
  @Input() user!: User;
  
  // Signal input (readonly)
  user = input.required<User>();
  
  // Optional signal input with default
  size = input<'small' | 'large'>('small');
  
  // Computed from signal input
  displayName = computed(() => {
    const user = this.user();
    return `${user.firstName} ${user.lastName}`;
  });
}
```

## Migration from RxJS

```typescript
// Before: RxJS
private countSubject = new BehaviorSubject(0);
count$ = this.countSubject.asObservable();

increment() {
  this.countSubject.next(this.countSubject.value + 1);
}

// After: Signals
count = signal(0);

increment() {
  this.count.update(c => c + 1);
}
```

## Best Practices

1. **Use `update()` over `set()` when modifying**: Ensures you don't miss current state
2. **Keep signals immutable**: Always create new objects/arrays
3. **Prefer computed over manual updates**: Let Angular track dependencies
4. **Use effects sparingly**: Mainly for side effects like logging or storage
5. **Name signals with nouns**: `count`, `user`, `items` (not `count$`)

## See Also

- [Signals vs RxJS](./signals-vs-rxjs.md)
- [Computed Values](./computed-values.md)
- [OnPush Change Detection](./onpush-change-detection.md)
- [Async Pipe Over Subscribe](./async-pipe-over-subscribe.md)
