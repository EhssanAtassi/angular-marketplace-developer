# Angular Signals Patterns

**Version:** Angular 16+  
**Status:** Stable (Developer Preview in 16, Stable in 17+)  
**Purpose:** Modern reactive state management without Zone.js overhead

---

## Core Concept

**Signals** are Angular's modern approach to reactive state management. Unlike observables, signals are synchronous, glitch-free, and optimized for change detection.

**Key Benefits:**
- ✅ Simpler mental model than RxJS
- ✅ Better performance (no Zone.js overhead)
- ✅ Fine-grained reactivity
- ✅ Type-safe
- ✅ Works seamlessly with OnPush change detection

---

## Signal Basics

### Creating Signals

```typescript
import { signal, computed, effect } from '@angular/core';

// Writable signal
const count = signal(0);               // number signal
const name = signal('John');           // string signal
const user = signal<User | null>(null); // object signal with null

// Read value (call as function)
console.log(count());  // 0
console.log(name());   // "John"

// Write value
count.set(5);          // Set to exact value
name.set('Jane');

// Update value (based on current)
count.update(n => n + 1);  // Increment
```

### Computed Signals

Computed signals automatically recalculate when dependencies change:

```typescript
const count = signal(0);

// Derived value
const double = computed(() => count() * 2);
const isEven = computed(() => count() % 2 === 0);
const message = computed(() => 
  `Count is ${count()} and ${isEven() ? 'even' : 'odd'}`
);

console.log(double());   // 0
console.log(message());  // "Count is 0 and even"

count.set(3);

console.log(double());   // 6
console.log(message());  // "Count is 3 and odd"
```

### Effects

Effects run side effects when signals change:

```typescript
import { effect } from '@angular/core';

const count = signal(0);

// Effect runs when count changes
effect(() => {
  console.log(`Count changed to: ${count()}`);
  localStorage.setItem('count', count().toString());
});

count.set(5);  // Logs: "Count changed to: 5"
```

---

## Pattern 1: Component State

### Basic Component State

```typescript
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-todo-list',
  standalone: true,
  template: `
    <input 
      [value]="newTodo()" 
      (input)="newTodo.set($any($event.target).value)"
    />
    <button (click)="addTodo()">Add</button>
    
    <p>Total: {{ total() }} | Active: {{ active() }} | Completed: {{ completed() }}</p>
    
    @for (todo of todos(); track todo.id) {
      <div>
        <input 
          type="checkbox" 
          [checked]="todo.completed"
          (change)="toggle(todo.id)"
        />
        <span [class.line-through]="todo.completed">
          {{ todo.text }}
        </span>
        <button (click)="remove(todo.id)">×</button>
      </div>
    }
  `
})
export class TodoListComponent {
  // State
  todos = signal<Todo[]>([]);
  newTodo = signal('');
  
  // Computed
  total = computed(() => this.todos().length);
  active = computed(() => this.todos().filter(t => !t.completed).length);
  completed = computed(() => this.todos().filter(t => t.completed).length);
  
  addTodo() {
    if (this.newTodo().trim()) {
      this.todos.update(todos => [
        ...todos,
        {
          id: Date.now().toString(),
          text: this.newTodo(),
          completed: false
        }
      ]);
      this.newTodo.set('');
    }
  }
  
  toggle(id: string) {
    this.todos.update(todos =>
      todos.map(t => t.id === id ? { ...t, completed: !t.completed } : t)
    );
  }
  
  remove(id: string) {
    this.todos.update(todos => todos.filter(t => t.id !== id));
  }
}
```

---

## Pattern 2: Derived State

### Computed from Multiple Signals

```typescript
@Component({...})
export class ShoppingCartComponent {
  items = signal<CartItem[]>([]);
  taxRate = signal(0.08);
  shippingCost = signal(5.99);
  
  // Computed from items
  subtotal = computed(() =>
    this.items().reduce((sum, item) => sum + item.price * item.quantity, 0)
  );
  
  // Computed from subtotal and taxRate
  tax = computed(() => this.subtotal() * this.taxRate());
  
  // Computed from multiple signals
  total = computed(() =>
    this.subtotal() + this.tax() + this.shippingCost()
  );
  
  // Computed boolean
  hasItems = computed(() => this.items().length > 0);
  canCheckout = computed(() => 
    this.hasItems() && this.total() > 0
  );
}
```

---

## Pattern 3: Signal Arrays

### Immutable Array Updates

```typescript
@Component({...})
export class ListComponent {
  items = signal<Item[]>([]);
  
  // Add item
  addItem(item: Item) {
    this.items.update(current => [...current, item]);
  }
  
  // Remove item
  removeItem(id: string) {
    this.items.update(current => current.filter(item => item.id !== id));
  }
  
  // Update item
  updateItem(id: string, updates: Partial<Item>) {
    this.items.update(current =>
      current.map(item => 
        item.id === id ? { ...item, ...updates } : item
      )
    );
  }
  
  // Sort items
  sortBy(key: keyof Item) {
    this.items.update(current => 
      [...current].sort((a, b) => a[key] > b[key] ? 1 : -1)
    );
  }
  
  // Filter items
  filteredItems = computed(() =>
    this.items().filter(item => item.active)
  );
}
```

---

## Pattern 4: Signal Objects

### Nested Object Updates

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  preferences: {
    theme: 'light' | 'dark';
    language: string;
  };
}

@Component({...})
export class UserProfileComponent {
  user = signal<User>({
    id: '1',
    name: 'John',
    email: 'john@example.com',
    preferences: {
      theme: 'light',
      language: 'en'
    }
  });
  
  // Update top-level property
  updateName(name: string) {
    this.user.update(u => ({ ...u, name }));
  }
  
  // Update nested property
  updateTheme(theme: 'light' | 'dark') {
    this.user.update(u => ({
      ...u,
      preferences: {
        ...u.preferences,
        theme
      }
    }));
  }
  
  // Computed from nested property
  isDarkMode = computed(() => this.user().preferences.theme === 'dark');
}
```

---

## Pattern 5: Loading States

### Common Loading Pattern

```typescript
interface LoadingState<T> {
  loading: boolean;
  data: T | null;
  error: string | null;
}

@Component({...})
export class DataComponent {
  private http = inject(HttpClient);
  
  state = signal<LoadingState<Product[]>>({
    loading: false,
    data: null,
    error: null
  });
  
  // Computed
  isLoading = computed(() => this.state().loading);
  hasError = computed(() => this.state().error !== null);
  hasData = computed(() => this.state().data !== null);
  products = computed(() => this.state().data ?? []);
  
  loadData() {
    this.state.update(s => ({ ...s, loading: true, error: null }));
    
    this.http.get<Product[]>('/api/products').subscribe({
      next: data => this.state.set({ loading: false, data, error: null }),
      error: err => this.state.set({ loading: false, data: null, error: err.message })
    });
  }
}
```

---

## Pattern 6: Form State

### Signal-Based Form

```typescript
@Component({
  selector: 'app-signup-form',
  template: `
    <form (submit)="handleSubmit($event)">
      <input 
        type="email"
        [value]="email()"
        (input)="email.set($any($event.target).value)"
        [class.invalid]="emailError()"
      />
      @if (emailError()) {
        <span class="error">{{ emailError() }}</span>
      }
      
      <input 
        type="password"
        [value]="password()"
        (input)="password.set($any($event.target).value)"
      />
      
      <button [disabled]="!isValid()">Sign Up</button>
    </form>
  `
})
export class SignupFormComponent {
  // Form fields
  email = signal('');
  password = signal('');
  
  // Validation
  emailError = computed(() => {
    const value = this.email();
    if (!value) return 'Email is required';
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
      return 'Invalid email format';
    }
    return null;
  });
  
  passwordError = computed(() => {
    const value = this.password();
    if (!value) return 'Password is required';
    if (value.length < 8) return 'Minimum 8 characters';
    return null;
  });
  
  isValid = computed(() =>
    !this.emailError() && !this.passwordError()
  );
  
  handleSubmit(event: Event) {
    event.preventDefault();
    if (this.isValid()) {
      console.log({ email: this.email(), password: this.password() });
    }
  }
}
```

---

## Pattern 7: Signal Effects

### Side Effects with Cleanup

```typescript
@Component({...})
export class AutoSaveComponent {
  content = signal('');
  
  constructor() {
    // Auto-save effect
    effect(() => {
      const data = this.content();
      
      if (data) {
        const timeoutId = setTimeout(() => {
          console.log('Auto-saving:', data);
          this.save(data);
        }, 1000);
        
        // Cleanup function
        return () => clearTimeout(timeoutId);
      }
    });
    
    // Log changes
    effect(() => {
      console.log('Content length:', this.content().length);
    });
  }
  
  save(data: string) {
    localStorage.setItem('draft', data);
  }
}
```

---

## Pattern 8: Signals with RxJS

### Converting Between Signals and Observables

```typescript
import { toSignal, toObservable } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';

@Component({...})
export class MixedComponent {
  // Observable to Signal
  private tick$ = interval(1000);
  tick = toSignal(this.tick$, { initialValue: 0 });
  
  // Signal to Observable
  count = signal(0);
  count$ = toObservable(this.count);
  
  constructor() {
    // Subscribe to signal as observable
    this.count$.subscribe(value => {
      console.log('Count changed:', value);
    });
  }
}
```

### Combining Signals with HTTP

```typescript
@Component({...})
export class UserComponent {
  private http = inject(HttpClient);
  userId = signal('123');
  
  // Convert signal to observable, then fetch
  user = toSignal(
    toObservable(this.userId).pipe(
      switchMap(id => this.http.get<User>(`/api/users/${id}`))
    ),
    { initialValue: null }
  );
}
```

---

## Pattern 9: Global State with Signals

### Signal-Based Store

```typescript
// store.service.ts
import { Injectable, signal, computed } from '@angular/core';

export interface AppState {
  user: User | null;
  theme: 'light' | 'dark';
  notifications: Notification[];
}

@Injectable({ providedIn: 'root' })
export class Store {
  // Private state
  private state = signal<AppState>({
    user: null,
    theme: 'light',
    notifications: []
  });
  
  // Public selectors
  user = computed(() => this.state().user);
  theme = computed(() => this.state().theme);
  notifications = computed(() => this.state().notifications);
  unreadCount = computed(() => 
    this.state().notifications.filter(n => !n.read).length
  );
  
  // Actions
  setUser(user: User | null) {
    this.state.update(s => ({ ...s, user }));
  }
  
  toggleTheme() {
    this.state.update(s => ({
      ...s,
      theme: s.theme === 'light' ? 'dark' : 'light'
    }));
  }
  
  addNotification(notification: Notification) {
    this.state.update(s => ({
      ...s,
      notifications: [...s.notifications, notification]
    }));
  }
}

// Usage in component
@Component({...})
export class AppComponent {
  private store = inject(Store);
  
  user = this.store.user;
  theme = this.store.theme;
  unreadCount = this.store.unreadCount;
  
  logout() {
    this.store.setUser(null);
  }
}
```

---

## Best Practices

### Do's ✅

1. **Use signals for synchronous state**
2. **Prefer computed over manual updates**
3. **Keep signals immutable** - always create new objects/arrays
4. **Use effects for side effects only**
5. **Combine with OnPush** change detection
6. **Use toSignal for observables** in components

### Don'ts ❌

1. **Don't mutate signal values directly**
   ```typescript
   // ❌ Bad
   items().push(newItem);
   
   // ✅ Good
   items.update(current => [...current, newItem]);
   ```

2. **Don't use effects for derived state**
   ```typescript
   // ❌ Bad - Use computed instead
   const count = signal(0);
   const double = signal(0);
   effect(() => double.set(count() * 2));
   
   // ✅ Good
   const double = computed(() => count() * 2);
   ```

3. **Don't create signals in loops**
4. **Don't read signals in constructors** (use ngOnInit or effects)

---

## Performance Tips

1. **Computed signals are cached** - only recalculate when dependencies change
2. **Signals trigger change detection only when value changes**
3. **Use OnPush** change detection with signals for best performance
4. **Signals are more efficient than observables** for synchronous state

---

## When to Use Signals vs RxJS

**Use Signals for:**
- Local component state
- Derived/computed values
- Form state
- UI state (loading, errors)

**Use RxJS for:**
- Asynchronous operations
- HTTP requests
- WebSocket streams
- Time-based operations (debounce, throttle)
- Complex async flows

**Use Both:**
- Convert observables to signals with `toSignal()`
- Convert signals to observables with `toObservable()`

---

## Summary

Signals provide:
- ✅ Simpler reactive state management
- ✅ Better performance
- ✅ Type safety
- ✅ Fine-grained reactivity
- ✅ Seamless integration with Angular

**Key Takeaway:** Signals are the future of Angular state management. Use them for synchronous state, computed values, and reactive UI updates.
