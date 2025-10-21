# Angular Change Detection Optimization

Comprehensive guide to Angular change detection, OnPush strategy, Signals, and Zone.js optimization.

## Table of Contents

1. [How Change Detection Works](#how-change-detection-works)
2. [Default vs OnPush](#default-vs-onpush)
3. [OnPush Strategy Patterns](#onpush-strategy-patterns)
4. [Angular Signals](#angular-signals)
5. [Zone.js Optimization](#zonejs-optimization)
6. [Manual Change Detection](#manual-change-detection)
7. [Common Pitfalls](#common-pitfalls)
8. [Performance Monitoring](#performance-monitoring)

---

## How Change Detection Works

### The Basics

Angular uses **Zone.js** to detect async operations and trigger change detection:

```typescript
// Any of these trigger change detection:
setTimeout(() => {});           // ✓ Triggers CD
setInterval(() => {}, 1000);    // ✓ Triggers CD
fetch('/api/data');             // ✓ Triggers CD
button.addEventListener('click'); // ✓ Triggers CD
Promise.resolve();              // ✓ Triggers CD
```

### The Component Tree

```
        App Component
           /    \
          /      \
    Header      Main
                /  \
               /    \
          Sidebar  Content
                    /  \
                   /    \
              List     Detail
```

**Default Strategy**: When CD runs, Angular checks **every component** in the tree.

### Change Detection Cycle

1. **Event occurs** (click, HTTP response, timer)
2. **Zone.js intercepts** the async operation
3. **Angular runs change detection** from root
4. **Each component** checks if bindings changed
5. **DOM updates** if changes detected

---

## Default vs OnPush

### Default Strategy

```typescript
@Component({
  selector: 'app-user-list',
  template: `
    <div>{{ currentTime }}</div>
    <button (click)="refresh()">Refresh</button>
  `
})
export class UserListComponent {
  currentTime = new Date().toLocaleTimeString();
  
  refresh() {
    this.currentTime = new Date().toLocaleTimeString();
  }
}
```

**Behavior**:
- Checks on **every** CD cycle
- Even if data didn't change
- Performance cost: **High** (scales with components)

### OnPush Strategy

```typescript
@Component({
  selector: 'app-user-list',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>{{ currentTime }}</div>
    <button (click)="refresh()">Refresh</button>
  `
})
export class UserListComponent {
  currentTime = new Date().toLocaleTimeString();
  
  refresh() {
    this.currentTime = new Date().toLocaleTimeString();
  }
}
```

**Behavior**:
- Only checks when:
  1. **@Input() reference changes**
  2. **Event handler** in component template
  3. **Observable emits** with async pipe
  4. **Manual detection** triggered
- Performance cost: **Low** (90% reduction)

### Performance Comparison

```typescript
// Scenario: 100 components, 10 CD cycles/second

// Default Strategy:
// 100 components × 10 cycles × 60 seconds = 60,000 checks/minute

// OnPush Strategy:
// ~5 components × 10 cycles × 60 seconds = 3,000 checks/minute
// ⚡ 95% reduction!
```

---

## OnPush Strategy Patterns

### Pattern 1: Immutable Data

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ProductListComponent {
  @Input() products: Product[] = [];
  
  // ❌ BAD: Mutates array (OnPush won't detect)
  addProductBad(product: Product) {
    this.products.push(product);
  }
  
  // ✅ GOOD: New array reference
  addProductGood(product: Product) {
    this.products = [...this.products, product];
  }
  
  // ✅ GOOD: Immutable update
  updateProduct(id: string, updates: Partial<Product>) {
    this.products = this.products.map(p =>
      p.id === id ? { ...p, ...updates } : p
    );
  }
}
```

### Pattern 2: Async Pipe

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <!-- ✅ Async pipe triggers OnPush automatically -->
    <div *ngFor="let user of users$ | async">
      {{ user.name }}
    </div>
  `
})
export class UserListComponent {
  users$ = this.userService.getUsers();
  
  constructor(private userService: UserService) {}
}
```

### Pattern 3: Event Handlers

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <button (click)="increment()">{{ count }}</button>
  `
})
export class CounterComponent {
  count = 0;
  
  // ✅ Event handlers in template trigger OnPush
  increment() {
    this.count++;
  }
}
```

### Pattern 4: Signals (Angular 16+)

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>{{ count() }}</div>
    <button (click)="increment()">+</button>
  `
})
export class CounterComponent {
  count = signal(0);
  
  increment() {
    this.count.update(c => c + 1);
  }
}
```

---

## Angular Signals

### Signal Basics

```typescript
// Create signal
const count = signal(0);

// Read signal
console.log(count()); // 0

// Update signal
count.set(5);
count.update(c => c + 1);

// Computed signal (derived state)
const doubled = computed(() => count() * 2);
```

### Signals in Components

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h2>Cart ({{ itemCount() }} items)</h2>
    <div>Total: {{ total() | currency }}</div>
    
    <div *ngFor="let item of items()">
      {{ item.name }} - {{ item.price | currency }}
      <button (click)="removeItem(item.id)">Remove</button>
    </div>
  `
})
export class CartComponent {
  items = signal<CartItem[]>([]);
  
  itemCount = computed(() => this.items().length);
  
  total = computed(() =>
    this.items().reduce((sum, item) => sum + item.price, 0)
  );
  
  addItem(item: CartItem) {
    this.items.update(items => [...items, item]);
  }
  
  removeItem(id: string) {
    this.items.update(items => items.filter(i => i.id !== id));
  }
}
```

### Signal Effects

```typescript
export class UserComponent {
  userId = signal<string | null>(null);
  user = signal<User | null>(null);
  
  constructor() {
    // Effect runs when dependencies change
    effect(() => {
      const id = this.userId();
      if (id) {
        this.userService.getUser(id).subscribe(
          user => this.user.set(user)
        );
      }
    });
  }
}
```

### Signals vs Observables

```typescript
// Observable approach
export class ProductsComponent {
  private searchTerm$ = new BehaviorSubject('');
  
  products$ = this.searchTerm$.pipe(
    debounceTime(300),
    switchMap(term => this.productService.search(term))
  );
  
  search(term: string) {
    this.searchTerm$.next(term);
  }
}

// Signal approach
export class ProductsComponent {
  searchTerm = signal('');
  
  products = computed(() => {
    // Note: computed is synchronous
    // For async, combine with effect
    return this.productService.search(this.searchTerm());
  });
  
  search(term: string) {
    this.searchTerm.set(term);
  }
}

// Hybrid approach (best for now)
export class ProductsComponent {
  searchTerm = signal('');
  
  products$ = toObservable(this.searchTerm).pipe(
    debounceTime(300),
    switchMap(term => this.productService.search(term))
  );
}
```

---

## Zone.js Optimization

### Run Outside Zone

```typescript
export class ChartComponent {
  constructor(private ngZone: NgZone) {}
  
  startAnimation() {
    // Run outside Angular's zone (no CD triggered)
    this.ngZone.runOutsideAngular(() => {
      const animate = () => {
        // Heavy animation logic
        this.updateChart();
        requestAnimationFrame(animate);
      };
      requestAnimationFrame(animate);
    });
  }
  
  // Manually trigger CD when needed
  updateData(data: any) {
    this.ngZone.run(() => {
      this.chartData = data;
    });
  }
}
```

### Polling Outside Zone

```typescript
export class LiveDataComponent implements OnInit, OnDestroy {
  data: any;
  private interval: any;
  
  constructor(private ngZone: NgZone) {}
  
  ngOnInit() {
    // Poll every second without triggering CD
    this.ngZone.runOutsideAngular(() => {
      this.interval = setInterval(() => {
        this.fetchData().then(data => {
          // Only trigger CD when data arrives
          this.ngZone.run(() => {
            this.data = data;
          });
        });
      }, 1000);
    });
  }
  
  ngOnDestroy() {
    clearInterval(this.interval);
  }
}
```

### Zone-less Angular (Experimental)

```typescript
// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideExperimentalZonelessChangeDetection()
  ]
});

// Component must use OnPush + Signals
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class MyComponent {
  count = signal(0); // Signals work without Zone.js
}
```

---

## Manual Change Detection

### ChangeDetectorRef

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ManualComponent {
  data: any;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  // Method 1: detectChanges() - Check this component
  updateData(data: any) {
    this.data = data;
    this.cdr.detectChanges();
  }
  
  // Method 2: markForCheck() - Mark for next CD cycle
  scheduleUpdate(data: any) {
    this.data = data;
    this.cdr.markForCheck();
  }
  
  // Method 3: detach() - Stop automatic CD
  ngOnInit() {
    this.cdr.detach();
    
    // Manual control
    setInterval(() => {
      this.data = new Date();
      this.cdr.detectChanges();
    }, 1000);
  }
  
  // Method 4: reattach() - Resume automatic CD
  enableAutoDetection() {
    this.cdr.reattach();
  }
}
```

### When to Use Manual Detection

```typescript
// ✅ Use Case 1: Third-party lib updates
export class MapComponent {
  constructor(private cdr: ChangeDetectorRef) {}
  
  initMap() {
    this.mapLibrary.on('update', (data) => {
      this.mapData = data;
      this.cdr.markForCheck(); // Tell Angular to check
    });
  }
}

// ✅ Use Case 2: WebSocket updates
export class LiveFeedComponent {
  constructor(
    private cdr: ChangeDetectorRef,
    private ws: WebSocketService
  ) {}
  
  ngOnInit() {
    this.ws.messages$.subscribe(msg => {
      this.messages.push(msg);
      this.cdr.markForCheck();
    });
  }
}

// ✅ Use Case 3: Performance-critical updates
export class GameComponent {
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    this.cdr.detach(); // Stop auto CD
    
    // Game loop
    const loop = () => {
      this.updateGame();
      
      // Only trigger CD when rendering
      if (this.shouldRender) {
        this.cdr.detectChanges();
      }
      
      requestAnimationFrame(loop);
    };
    requestAnimationFrame(loop);
  }
}
```

---

## Common Pitfalls

### Pitfall 1: Mutating @Input

```typescript
// ❌ BAD: Parent change won't trigger OnPush
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ChildComponent {
  @Input() user: User;
}

// Parent mutates object
this.user.name = 'Updated'; // OnPush child won't detect

// ✅ GOOD: Create new reference
this.user = { ...this.user, name: 'Updated' };
```

### Pitfall 2: Nested Objects

```typescript
// ❌ BAD: Nested mutation
@Input() config = { theme: { color: 'blue' } };

this.config.theme.color = 'red'; // Won't trigger OnPush

// ✅ GOOD: Deep immutable update
this.config = {
  ...this.config,
  theme: { ...this.config.theme, color: 'red' }
};
```

### Pitfall 3: Array Modifications

```typescript
// ❌ BAD: Mutating array
@Input() items: string[] = [];

this.items.push('new');    // Won't trigger
this.items.splice(0, 1);   // Won't trigger
this.items[0] = 'updated'; // Won't trigger

// ✅ GOOD: New array
this.items = [...this.items, 'new'];
this.items = this.items.filter((_, i) => i !== 0);
this.items = this.items.map((item, i) => i === 0 ? 'updated' : item);
```

### Pitfall 4: Async without async pipe

```typescript
// ❌ BAD: Manual subscription in OnPush
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class BadComponent {
  users: User[] = [];
  
  ngOnInit() {
    this.userService.getUsers().subscribe(users => {
      this.users = users; // Won't trigger OnPush!
    });
  }
}

// ✅ GOOD: Use async pipe
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<div *ngFor="let user of users$ | async">...</div>`
})
export class GoodComponent {
  users$ = this.userService.getUsers();
}

// ✅ GOOD: Or use markForCheck
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AlsoGoodComponent {
  users: User[] = [];
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    this.userService.getUsers().subscribe(users => {
      this.users = users;
      this.cdr.markForCheck();
    });
  }
}
```

---

## Performance Monitoring

### Chrome DevTools Profiler

```typescript
// 1. Open Chrome DevTools
// 2. Performance tab
// 3. Click Record
// 4. Interact with app
// 5. Stop recording

// Look for:
// - Long tasks (>50ms)
// - Excessive change detection
// - Layout thrashing
```

### Angular DevTools

```typescript
// 1. Install Angular DevTools extension
// 2. Open DevTools > Angular tab
// 3. Profiler section
// 4. Record profile
// 5. Analyze change detection cycles

// Shows:
// - Change detection timing
// - Component tree
// - Change detection strategy per component
```

### Performance Marks

```typescript
export class ProfiledComponent {
  loadData() {
    performance.mark('data-load-start');
    
    this.service.getData().subscribe(data => {
      this.data = data;
      
      performance.mark('data-load-end');
      performance.measure(
        'Data Load',
        'data-load-start',
        'data-load-end'
      );
      
      // View in Performance tab
      const measure = performance.getEntriesByName('Data Load')[0];
      console.log(`Data load took ${measure.duration}ms`);
    });
  }
}
```

### Change Detection Counter

```typescript
// Directive to count CD cycles
@Directive({
  selector: '[cdCounter]',
  standalone: true
})
export class CdCounterDirective implements DoCheck {
  private count = 0;
  
  ngDoCheck() {
    this.count++;
    console.log(`CD cycle #${this.count}`);
  }
}

// Usage
<app-my-component cdCounter></app-my-component>
```

---

## Best Practices Summary

✅ **DO**:
- Use `ChangeDetectionStrategy.OnPush` by default
- Use `async` pipe for observables
- Use Angular Signals for reactive state
- Create new references for objects/arrays
- Use `trackBy` for lists
- Run heavy operations outside Zone
- Use `markForCheck()` for third-party integrations

❌ **DON'T**:
- Mutate `@Input()` properties
- Manually subscribe in OnPush without `markForCheck()`
- Use Default strategy unless necessary
- Perform heavy computations in getters
- Forget to unsubscribe from observables
- Mix imperative and reactive patterns

---

## Quick Reference

### Change Detection Triggers (OnPush)

```typescript
✓ @Input() reference changes
✓ Event from template
✓ async pipe emission
✓ cdr.detectChanges()
✓ cdr.markForCheck()
✓ Signal updates (in templates)

✗ Object mutation
✗ Array.push/splice/pop
✗ Nested property changes
✗ Manual subscription
✗ setTimeout/setInterval (without events)
```

### Strategy Comparison

| Feature | Default | OnPush |
|---------|---------|--------|
| Checks on every CD | ✓ | ✗ |
| Checks on @Input change | ✓ | ✓ (reference) |
| Checks on events | ✓ | ✓ |
| Checks on async pipe | ✓ | ✓ |
| Performance | Low | High |
| Complexity | Low | Medium |

---

*Master change detection for blazing-fast Angular apps! 🚀*
