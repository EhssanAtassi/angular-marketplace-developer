# Optimize Component Command

Analyze and optimize Angular component performance using OnPush, trackBy, memoization, and other strategies.

## Usage

```bash
/angular-performance:optimize-component <ComponentName>
```

## Natural Language

- "Optimize UserListComponent performance"
- "Make my dashboard component faster"
- "Reduce change detection in ProductCardComponent"

## Optimization Strategies

### 1. OnPush Change Detection

```typescript
// Before
@Component({
  selector: 'app-product-list',
  templateUrl: './product-list.component.html'
})
export class ProductListComponent {
  @Input() products: Product[] = [];
}

// After
@Component({
  selector: 'app-product-list',
  changeDetection: ChangeDetectionStrategy.OnPush,
  templateUrl: './product-list.component.html'
})
export class ProductListComponent {
  @Input() products: Product[] = [];
}
```

**Impact**: Reduces change detection checks by 80-90%

### 2. TrackBy Functions

```typescript
// Before
<div *ngFor="let product of products">
  {{ product.name }}
</div>

// After
@for (product of products; track product.id) {
  <div>{{ product.name }}</div>
}

// Or with trackBy function
trackById(index: number, item: Product): string {
  return item.id;
}

<div *ngFor="let product of products; trackBy: trackById">
  {{ product.name }}
</div>
```

**Impact**: Prevents unnecessary DOM recreations, 60% faster re-renders

### 3. Memoization with Signals

```typescript
// Before: Computed on every change detection
get filteredProducts(): Product[] {
  return this.products().filter(p => p.price > 100);
}

// After: Memoized with computed signal
filteredProducts = computed(() => 
  this.products().filter(p => p.price > 100)
);
```

**Impact**: Computation runs only when dependencies change

### 4. Pure Pipes

```typescript
// Before: Impure pipe (runs on every CD)
@Pipe({ name: 'filter' })
export class FilterPipe {
  transform(items: any[], searchTerm: string): any[] {
    return items.filter(item => 
      item.name.includes(searchTerm)
    );
  }
}

// After: Pure pipe with immutable data
@Pipe({ name: 'filter', pure: true })
export class FilterPipe {
  transform(items: any[], searchTerm: string): any[] {
    return items.filter(item => 
      item.name.includes(searchTerm)
    );
  }
}

// Component: Use immutable operations
addProduct(product: Product) {
  this.products = [...this.products, product]; // ✅ New reference
  // NOT: this.products.push(product); // ❌ Mutates array
}
```

### 5. Virtual Scrolling

```typescript
// Before: Renders all items
<div *ngFor="let user of users">
  <app-user-card [user]="user" />
</div>

// After: Renders only visible items
<cdk-virtual-scroll-viewport itemSize="80" class="h-screen">
  <app-user-card 
    *cdkVirtualFor="let user of users; trackBy: trackById"
    [user]="user"
  />
</cdk-virtual-scroll-viewport>
```

**Impact**: 95% faster rendering for large lists (>1000 items)

### 6. Lazy Loading Images

```typescript
// Before
<img [src]="product.imageUrl" [alt]="product.name">

// After
<img 
  [src]="product.imageUrl" 
  [alt]="product.name"
  loading="lazy"
  [width]="300"
  [height]="200"
>
```

### 7. Subscription Management

```typescript
// Before: Manual unsubscribe
export class UserComponent implements OnInit, OnDestroy {
  private subscription = new Subscription();
  
  ngOnInit() {
    this.subscription.add(
      this.userService.getUser().subscribe(...)
    );
  }
  
  ngOnDestroy() {
    this.subscription.unsubscribe();
  }
}

// After: takeUntilDestroyed()
export class UserComponent implements OnInit {
  private destroyRef = inject(DestroyRef);
  
  ngOnInit() {
    this.userService.getUser()
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(...);
  }
}

// Or use async pipe (best)
user$ = this.userService.getUser();
// Template: {{ user$ | async }}
```

### 8. Detach Change Detection

```typescript
// For components that rarely update
export class StaticContentComponent implements OnInit {
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    this.cdr.detach(); // Stop automatic change detection
  }
  
  updateContent() {
    // Manually trigger when needed
    this.cdr.detectChanges();
  }
}
```

## Optimization Checklist

```typescript
// Component optimization checklist
@Component({
  selector: 'app-optimized',
  templateUrl: './optimized.component.html',
  styleUrls: ['./optimized.component.scss'],
  changeDetection: ChangeDetectionStrategy.OnPush, // ✅ 1. OnPush
  standalone: true,
  imports: [CommonModule, CdkVirtualScrollViewport]
})
export class OptimizedComponent {
  // ✅ 2. Use signals for reactive state
  private userService = inject(UserService);
  users = signal<User[]>([]);
  
  // ✅ 3. Computed for derived state
  activeUsers = computed(() => 
    this.users().filter(u => u.active)
  );
  
  // ✅ 4. TrackBy function
  trackById(index: number, user: User): string {
    return user.id;
  }
  
  ngOnInit() {
    // ✅ 5. takeUntilDestroyed for subscriptions
    this.userService.getUsers()
      .pipe(takeUntilDestroyed())
      .subscribe(users => this.users.set(users));
  }
}
```

```html
<!-- Template optimizations -->
<cdk-virtual-scroll-viewport itemSize="60" class="h-screen">
  <!-- ✅ 6. Virtual scrolling -->
  <div *cdkVirtualFor="let user of users(); trackBy: trackById">
    <!-- ✅ 7. Lazy loading images -->
    <img 
      [src]="user.avatar" 
      loading="lazy"
      [width]="50" 
      [height]="50"
    >
    
    <!-- ✅ 8. Async pipe for observables -->
    <span>{{ user.name }}</span>
  </div>
</cdk-virtual-scroll-viewport>
```

## Performance Metrics

Measure with Chrome DevTools:

```typescript
// Add performance marks
performance.mark('component-init-start');
// ... component logic
performance.mark('component-init-end');
performance.measure('Component Init', 'component-init-start', 'component-init-end');
```

## Output Example

```
🎯 Component Optimization Report: UserListComponent

📊 Before:
- Change Detection: Default (runs on every event)
- List Rendering: 2,500 items fully rendered
- Memory Usage: 145 MB
- Initial Render: 850ms
- FPS during scroll: 25-30

✅ After Optimizations:
1. ✓ OnPush change detection
2. ✓ trackBy function for list
3. ✓ Virtual scrolling (renders ~20 items)
4. ✓ Image lazy loading
5. ✓ Subscription cleanup with takeUntilDestroyed

📈 Results:
- Change Detection: 92% reduction in checks
- List Rendering: 2,500 → 20 items (99% less DOM)
- Memory Usage: 145 MB → 28 MB (81% reduction)
- Initial Render: 850ms → 42ms (95% faster)
- FPS during scroll: 58-60 (smooth)

🎁 Bonus:
- Lighthouse Performance Score: 67 → 98
- First Contentful Paint: 2.1s → 0.8s
- Time to Interactive: 3.5s → 1.2s
```

## Common Patterns

### Pattern 1: List with Filters

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <input [(ngModel)]="searchTerm" placeholder="Search...">
    
    <cdk-virtual-scroll-viewport itemSize="60">
      <div *cdkVirtualFor="let item of filteredItems(); trackBy: trackById">
        {{ item.name }}
      </div>
    </cdk-virtual-scroll-viewport>
  `
})
export class FilterableListComponent {
  items = signal<Item[]>([]);
  searchTerm = signal('');
  
  filteredItems = computed(() => {
    const term = this.searchTerm().toLowerCase();
    return this.items().filter(item => 
      item.name.toLowerCase().includes(term)
    );
  });
  
  trackById = (index: number, item: Item) => item.id;
}
```

### Pattern 2: Nested Components

```typescript
// Parent: Smart component with OnPush
@Component({
  selector: 'app-user-dashboard',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <app-user-profile [user]="user()" />
    <app-user-stats [stats]="stats()" />
  `
})
export class UserDashboardComponent {
  user = signal<User | null>(null);
  stats = signal<Stats | null>(null);
}

// Child: Dumb component with OnPush
@Component({
  selector: 'app-user-profile',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h2>{{ user.name }}</h2>
    <img [src]="user.avatar" loading="lazy">
  `
})
export class UserProfileComponent {
  @Input({ required: true }) user!: User;
}
```

---

*Optimize smart, measure always! 🚀*
