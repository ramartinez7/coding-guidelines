# TrackBy Functions

> Always provide trackBy functions for *ngFor to optimize list rendering and prevent unnecessary DOM recreation.

## Problem

Without a `trackBy` function, Angular uses object identity to track list items. When the list is updated, Angular recreates all DOM elements even if the data hasn't changed, causing performance issues and losing component state.

## Example

### ❌ Before

```typescript
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users">
      <input [(ngModel)]="user.name">
      <span>{{ user.email }}</span>
    </div>
  `
})
export class UserListComponent {
  users: User[] = [];
  
  refreshUsers() {
    // Fetching new array from API
    this.http.get<User[]>('/api/users').subscribe(users => {
      this.users = users;  // New array reference!
      // Angular recreates ALL DOM elements, even for unchanged users
      // User input is lost, animations restart, etc.
    });
  }
}
```

### ✅ After

```typescript
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users; trackBy: trackById">
      <input [(ngModel)]="user.name">
      <span>{{ user.email }}</span>
    </div>
  `
})
export class UserListComponent {
  users: User[] = [];
  
  refreshUsers() {
    this.http.get<User[]>('/api/users').subscribe(users => {
      this.users = users;
      // Angular reuses DOM for items with same ID
      // Only creates/updates DOM for changed items
    });
  }
  
  trackById(index: number, user: User): number {
    return user.id;  // Use unique identifier
  }
}
```

## Why It's a Problem

1. **Performance**: Recreating DOM is expensive, especially with complex components or large lists.

2. **Lost state**: Component state (input values, expanded sections) is lost when DOM is recreated.

3. **Animation restart**: CSS animations restart from beginning when elements are recreated.

4. **Focus loss**: Active element loses focus when recreated.

5. **Unnecessary re-initialization**: Component lifecycle hooks (`ngOnInit`, etc.) run again unnecessarily.

## Symptoms

- Input fields losing their values when parent list updates
- Animations restarting unexpectedly
- Poor performance when updating lists
- Components "flickering" or reinitializing on list updates
- Focus jumping to unexpected elements
- Slow scrolling in long lists

## Benefits

- **Better performance**: Only update changed elements, not entire list
- **Preserved state**: Input values and component state maintained
- **Smooth animations**: Animations continue without restart
- **Maintained focus**: Active element stays focused
- **Reduced initialization**: Component lifecycle runs only when needed

## How TrackBy Works

```typescript
trackById(index: number, item: User): number {
  return item.id;
  // Angular uses this return value to identify items:
  // - Same ID = reuse existing DOM element (update it)
  // - Different ID = create new DOM element
  // - Missing ID = remove DOM element
}
```

Angular compares the tracking values before and after update:

```typescript
// Before: [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }]
// After:  [{ id: 1, name: 'Alice' }, { id: 2, name: 'Robert' }, { id: 3, name: 'Charlie' }]

// Results:
// - id: 1 → reuse DOM, no changes needed
// - id: 2 → reuse DOM, update name from 'Bob' to 'Robert'
// - id: 3 → create new DOM element
```

## Common TrackBy Patterns

### By Unique ID

```typescript
@Component({
  template: `
    <div *ngFor="let item of items; trackBy: trackById">
      {{ item.name }}
    </div>
  `
})
export class Component {
  trackById(index: number, item: Item): number {
    return item.id;
  }
}
```

### By Index (Immutable Lists Only)

```typescript
// Only use if list items never change order or content
trackByIndex(index: number): number {
  return index;
}
```

⚠️ **Warning**: Don't use `trackByIndex` if list items can be reordered, inserted, or removed from middle. It defeats the purpose of trackBy.

### By Composite Key

```typescript
// For items without single unique identifier
trackByComposite(index: number, item: OrderItem): string {
  return `${item.orderId}-${item.productId}`;
}
```

### By Reference (Immutable Data)

```typescript
// Works if you use immutable data patterns
trackByReference(index: number, item: User): User {
  return item;  // Object identity used as key
}
```

## Advanced Example: Reusable TrackBy Service

```typescript
// track-by.service.ts
@Injectable({ providedIn: 'root' })
export class TrackByService {
  // Generic trackBy by property name
  byProperty<T>(property: keyof T) {
    return (index: number, item: T) => item[property];
  }
  
  // TrackBy by id specifically
  byId<T extends { id: any }>(index: number, item: T) {
    return item.id;
  }
  
  // TrackBy by index
  byIndex(index: number) {
    return index;
  }
}

// Usage in component
@Component({
  template: `
    <div *ngFor="let user of users; trackBy: trackBy.byId">
      {{ user.name }}
    </div>
  `
})
export class UserListComponent {
  users: User[] = [];
  
  constructor(public trackBy: TrackByService) {}
}
```

## Performance Comparison

```typescript
// Without trackBy: 1000 items, update 10 items
// - Destroys: 1000 DOM elements
// - Creates: 1000 DOM elements
// - Time: ~500ms

// With trackBy: 1000 items, update 10 items
// - Destroys: 0 DOM elements
// - Creates: 0 DOM elements
// - Updates: 10 DOM elements
// - Time: ~20ms (25x faster!)
```

## Common Mistakes

### Using Index with Mutable Lists

```typescript
// ❌ BAD: Items can be reordered
@Component({
  template: `
    <div *ngFor="let item of items; trackBy: trackByIndex">
      {{ item.name }}
    </div>
  `
})
export class Component {
  items: Item[] = [];
  
  sortItems() {
    this.items.sort((a, b) => a.name.localeCompare(b.name));
    // trackByIndex causes Angular to update wrong elements!
  }
  
  trackByIndex(index: number): number {
    return index;
  }
}
```

### Returning Unstable Values

```typescript
// ❌ BAD: Returns different value each time for same item
trackByRandom(index: number, item: User): number {
  return Math.random(); // Never do this!
}

// ❌ BAD: Returns different object each time
trackByObject(index: number, item: User): object {
  return { id: item.id }; // New object each time
}
```

### Not Using TrackBy at All

```typescript
// ❌ BAD: Missing trackBy
<div *ngFor="let item of items">{{ item.name }}</div>

// ✅ GOOD: Always include trackBy
<div *ngFor="let item of items; trackBy: trackById">{{ item.name }}</div>
```

## When TrackBy Doesn't Matter

TrackBy is less important when:

1. List never changes after initial render
2. List is very small (< 10 items)
3. List items are simple text with no state
4. Performance isn't a concern

Even in these cases, it's good practice to include trackBy.

## Testing TrackBy

```typescript
describe('UserListComponent', () => {
  it('should use trackBy function', () => {
    const component = new UserListComponent();
    const user = { id: 1, name: 'Test' };
    
    expect(component.trackById(0, user)).toBe(1);
  });
  
  it('should preserve input values when list updates', () => {
    const fixture = TestBed.createComponent(UserListComponent);
    const compiled = fixture.nativeElement;
    
    // Render initial list
    fixture.componentInstance.users = [
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' }
    ];
    fixture.detectChanges();
    
    // User types in input
    const input = compiled.querySelector('input') as HTMLInputElement;
    input.value = 'Modified';
    input.dispatchEvent(new Event('input'));
    
    // Update list (same items, new array)
    fixture.componentInstance.users = [
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' }
    ];
    fixture.detectChanges();
    
    // Input value should be preserved
    expect(input.value).toBe('Modified');
  });
});
```

## See Also

- [OnPush Change Detection](./onpush-change-detection.md)
- [Virtual Scrolling](./virtual-scrolling.md)
- [Immutable Data Patterns](./immutable-data-patterns.md)
