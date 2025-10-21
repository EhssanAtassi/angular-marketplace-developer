# RxJS Operators for Angular

**Purpose:** Essential RxJS operators every Angular developer should master  
**Level:** Intermediate to Advanced  
**Version:** RxJS 7+

---

## Core Concepts

**Operators** are functions that enable composing asynchronous operations with observables. They transform, filter, combine, and manage observable streams.

**Key Principles:**
- Operators are **pure functions**
- They **don't modify** the source observable
- They **return a new** observable
- They can be **chained** together

---

## Category 1: Transformation Operators

### map

**Purpose:** Transform each value emitted

```typescript
import { of } from 'rxjs';
import { map } from 'rxjs/operators';

// Example 1: Simple transformation
of(1, 2, 3).pipe(
  map(x => x * 10)
).subscribe(console.log);
// Output: 10, 20, 30

// Example 2: Object transformation
interface User { id: number; name: string; }
interface UserDisplay { id: number; displayName: string; }

this.users$.pipe(
  map((users: User[]) => users.map(u => ({
    id: u.id,
    displayName: u.name.toUpperCase()
  })))
);
```

### switchMap

**Purpose:** Switch to a new observable, canceling previous

**Use when:** Making HTTP requests based on user input

```typescript
import { switchMap } from 'rxjs/operators';

// Search as user types
this.searchTerm$.pipe(
  debounceTime(300),
  switchMap(term => this.http.get(`/api/search?q=${term}`))
).subscribe(results => console.log(results));

// Load user details when ID changes
this.userId$.pipe(
  switchMap(id => this.http.get(`/api/users/${id}`))
).subscribe(user => this.user.set(user));
```

**Why switchMap?** Automatically cancels previous HTTP request if new search term arrives.

### mergeMap (flatMap)

**Purpose:** Merge all inner observables

**Use when:** You want all requests to complete, not cancel previous

```typescript
import { mergeMap } from 'rxjs/operators';

// Send analytics for each click (don't cancel)
this.clicks$.pipe(
  mergeMap(event => this.analytics.track(event))
).subscribe();

// Process multiple files in parallel
this.files$.pipe(
  mergeMap(file => this.uploadFile(file))
).subscribe(result => console.log('Uploaded:', result));
```

### concatMap

**Purpose:** Process observables in order, wait for each to complete

**Use when:** Order matters (e.g., sequential API calls)

```typescript
import { concatMap } from 'rxjs/operators';

// Process queue in order
this.queue$.pipe(
  concatMap(task => this.processTask(task))
).subscribe(result => console.log('Processed:', result));

// Sequential API calls
this.users$.pipe(
  concatMap(user => this.http.post('/api/users', user))
).subscribe();
```

### exhaustMap

**Purpose:** Ignore new values while current is processing

**Use when:** Prevent duplicate submissions

```typescript
import { exhaustMap } from 'rxjs/operators';

// Prevent double-click on submit button
this.submitClick$.pipe(
  exhaustMap(() => this.http.post('/api/form', this.formData))
).subscribe();

// Login button (ignore clicks while logging in)
this.loginAttempt$.pipe(
  exhaustMap(credentials => this.auth.login(credentials))
).subscribe();
```

---

## Category 2: Filtering Operators

### filter

**Purpose:** Emit only values that pass a condition

```typescript
import { filter } from 'rxjs/operators';

// Only even numbers
of(1, 2, 3, 4, 5).pipe(
  filter(x => x % 2 === 0)
).subscribe(console.log);
// Output: 2, 4

// Only non-null users
this.user$.pipe(
  filter(user => user !== null)
).subscribe(user => console.log(user.name));

// Only valid emails
this.emailInput$.pipe(
  filter(email => this.isValidEmail(email))
).subscribe(email => this.checkAvailability(email));
```

### debounceTime

**Purpose:** Wait for silence before emitting

**Use when:** Search input, window resize

```typescript
import { debounceTime } from 'rxjs/operators';

// Wait 300ms after user stops typing
this.searchInput$.pipe(
  debounceTime(300),
  switchMap(term => this.search(term))
).subscribe(results => this.results.set(results));

// Window resize handler
fromEvent(window, 'resize').pipe(
  debounceTime(200)
).subscribe(() => this.handleResize());
```

### throttleTime

**Purpose:** Emit first value, then ignore for duration

**Use when:** Scroll events, rapid clicks

```typescript
import { throttleTime } from 'rxjs/operators';

// Handle scroll at most once per 100ms
fromEvent(window, 'scroll').pipe(
  throttleTime(100)
).subscribe(() => this.checkScrollPosition());

// Rate-limit button clicks
this.buttonClick$.pipe(
  throttleTime(1000)
).subscribe(() => this.handleClick());
```

### distinctUntilChanged

**Purpose:** Only emit when value changes

```typescript
import { distinctUntilChanged } from 'rxjs/operators';

// Only emit when search term actually changes
this.searchInput$.pipe(
  distinctUntilChanged(),
  switchMap(term => this.search(term))
).subscribe();

// Only emit when user ID changes
this.userId$.pipe(
  distinctUntilChanged(),
  switchMap(id => this.loadUser(id))
).subscribe();
```

### take / takeUntil

**Purpose:** Take specific number or until condition

```typescript
import { take, takeUntil } from 'rxjs/operators';

// Take first 5 values
this.stream$.pipe(
  take(5)
).subscribe();

// Take until component destroyed
private destroy$ = new Subject<void>();

this.data$.pipe(
  takeUntil(this.destroy$)
).subscribe(data => console.log(data));

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}

// Modern approach with takeUntilDestroyed
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

this.data$.pipe(
  takeUntilDestroyed()
).subscribe(data => console.log(data));
```

---

## Category 3: Combination Operators

### combineLatest

**Purpose:** Emit when ANY source emits (after all emit at least once)

**Use when:** Combining multiple form fields, filters

```typescript
import { combineLatest } from 'rxjs';

// Wait for both user and settings to load
combineLatest([
  this.user$,
  this.settings$
]).pipe(
  map(([user, settings]) => ({ user, settings }))
).subscribe(data => console.log(data));

// Combine multiple filters
combineLatest([
  this.searchTerm$,
  this.category$,
  this.priceRange$
]).pipe(
  map(([search, category, price]) => ({
    search, category, price
  })),
  switchMap(filters => this.fetchProducts(filters))
).subscribe(products => this.products.set(products));
```

### forkJoin

**Purpose:** Emit when ALL sources complete (like Promise.all)

**Use when:** Loading multiple independent resources

```typescript
import { forkJoin } from 'rxjs';

// Load multiple resources on init
forkJoin({
  user: this.http.get('/api/user'),
  config: this.http.get('/api/config'),
  permissions: this.http.get('/api/permissions')
}).subscribe(({ user, config, permissions }) => {
  this.initialize(user, config, permissions);
});

// Parallel API calls
forkJoin([
  this.http.get('/api/products'),
  this.http.get('/api/categories'),
  this.http.get('/api/brands')
]).subscribe(([products, categories, brands]) => {
  // All loaded
});
```

### merge

**Purpose:** Emit from any source as soon as it emits

**Use when:** Combining event streams

```typescript
import { merge } from 'rxjs';

// Combine multiple event sources
merge(
  this.clicks$,
  this.hovers$,
  this.focuses$
).subscribe(event => this.trackEvent(event));

// Combine refresh triggers
merge(
  this.manualRefresh$,
  this.autoRefresh$,
  this.dataChanged$
).pipe(
  switchMap(() => this.loadData())
).subscribe();
```

### withLatestFrom

**Purpose:** Combine with latest value from other observables

**Use when:** Need secondary data with primary stream

```typescript
import { withLatestFrom } from 'rxjs/operators';

// Submit form with latest user data
this.submitButton$.pipe(
  withLatestFrom(this.form$, this.user$),
  map(([_, formData, user]) => ({ formData, user }))
).subscribe(({ formData, user }) => {
  this.submit(formData, user);
});

// Apply filter with latest settings
this.searchTerm$.pipe(
  withLatestFrom(this.filters$, this.sortOrder$),
  switchMap(([term, filters, sort]) => 
    this.search(term, filters, sort)
  )
).subscribe();
```

---

## Category 4: Error Handling

### catchError

**Purpose:** Catch errors and return fallback observable

```typescript
import { catchError } from 'rxjs/operators';
import { of, EMPTY } from 'rxjs';

// Return empty array on error
this.http.get('/api/data').pipe(
  catchError(error => {
    console.error('Failed to load:', error);
    return of([]);  // Fallback value
  })
).subscribe(data => console.log(data));

// Return empty observable (complete immediately)
this.http.get('/api/data').pipe(
  catchError(() => EMPTY)
).subscribe();

// Re-throw after logging
this.http.get('/api/data').pipe(
  catchError(error => {
    console.error(error);
    return throwError(() => error);
  })
).subscribe();
```

### retry

**Purpose:** Retry failed observable

```typescript
import { retry } from 'rxjs/operators';

// Retry 3 times on failure
this.http.get('/api/data').pipe(
  retry(3),
  catchError(error => {
    console.error('Failed after 3 retries');
    return of([]);
  })
).subscribe();

// Retry with delay (RxJS 7+)
import { retry } from 'rxjs/operators';

this.http.get('/api/data').pipe(
  retry({
    count: 3,
    delay: 1000  // Wait 1s between retries
  })
).subscribe();
```

---

## Category 5: Utility Operators

### tap

**Purpose:** Perform side effects without modifying stream

```typescript
import { tap } from 'rxjs/operators';

// Log for debugging
this.http.get('/api/data').pipe(
  tap(data => console.log('Received:', data)),
  map(data => data.items),
  tap(items => console.log('Mapped:', items))
).subscribe();

// Track analytics
this.searchTerm$.pipe(
  tap(term => this.analytics.track('search', { term })),
  switchMap(term => this.search(term))
).subscribe();

// Update loading state
this.loadData().pipe(
  tap(() => this.loading.set(true)),
  finalize(() => this.loading.set(false))
).subscribe();
```

### shareReplay

**Purpose:** Share observable and replay values to new subscribers

```typescript
import { shareReplay } from 'rxjs/operators';

// Cache HTTP request
private config$ = this.http.get('/api/config').pipe(
  shareReplay(1)  // Cache last value
);

// Multiple subscribers get same value
this.config$.subscribe(config => console.log('Sub 1:', config));
this.config$.subscribe(config => console.log('Sub 2:', config));
// Only one HTTP request made!
```

### finalize

**Purpose:** Execute code when observable completes or errors

```typescript
import { finalize } from 'rxjs/operators';

// Always hide loading spinner
this.loadData().pipe(
  tap(() => this.loading.set(true)),
  finalize(() => this.loading.set(false))
).subscribe({
  next: data => console.log(data),
  error: err => console.error(err)
  // loading.set(false) runs regardless
});
```

---

## Common Patterns

### Pattern 1: Search with Debounce

```typescript
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.http.get(`/api/search?q=${term}`)),
  catchError(() => of([]))
).subscribe(results => this.results.set(results));
```

### Pattern 2: Auto-Save

```typescript
this.form.valueChanges.pipe(
  debounceTime(2000),
  distinctUntilChanged(),
  tap(() => this.saving.set(true)),
  switchMap(value => this.http.put('/api/save', value)),
  finalize(() => this.saving.set(false))
).subscribe();
```

### Pattern 3: Polling

```typescript
import { interval, switchMap } from 'rxjs';

interval(5000).pipe(
  switchMap(() => this.http.get('/api/status')),
  catchError(() => of(null))
).subscribe(status => this.status.set(status));
```

### Pattern 4: Type-ahead with Minimum Length

```typescript
this.searchInput$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  filter(term => term.length >= 3),
  switchMap(term => this.search(term)),
  catchError(() => of([]))
).subscribe(results => this.results.set(results));
```

---

## Decision Tree

**Need to transform values?** → `map`  
**Need to switch to new observable?** → `switchMap`  
**Need to wait for all to complete?** → `forkJoin`  
**Need to combine latest values?** → `combineLatest`  
**Need to filter values?** → `filter`  
**Need to handle errors?** → `catchError`  
**Need to retry?** → `retry`  
**Need to share result?** → `shareReplay`  
**Need to debounce?** → `debounceTime`  
**Need to throttle?** → `throttleTime`

---

## Best Practices

1. **Always unsubscribe** - Use `takeUntilDestroyed()` or `async` pipe
2. **Prefer `async` pipe** over manual subscriptions
3. **Use `switchMap`** for dependent HTTP requests
4. **Use `forkJoin`** for parallel independent requests
5. **Add error handling** with `catchError`
6. **Cache with `shareReplay`** for expensive operations
7. **Debounce user input** with `debounceTime`
8. **Log with `tap`** for debugging

---

## Common Mistakes

**❌ Memory leaks:**
```typescript
// Bad - no unsubscribe
this.data$.subscribe(data => console.log(data));
```

**✅ Fixed:**
```typescript
// Good - auto unsubscribe
this.data$.pipe(
  takeUntilDestroyed()
).subscribe(data => console.log(data));

// Or use async pipe
template: `{{ data$ | async }}`
```

**❌ Nested subscriptions:**
```typescript
// Bad - pyramid of doom
this.users$.subscribe(users => {
  this.http.get('/api/settings').subscribe(settings => {
    // ...
  });
});
```

**✅ Fixed:**
```typescript
// Good - use switchMap
this.users$.pipe(
  switchMap(users => this.http.get('/api/settings').pipe(
    map(settings => ({ users, settings }))
  ))
).subscribe(({ users, settings }) => {
  // ...
});
```

---

## Summary

Master these operators:
- **Transformation**: `map`, `switchMap`, `mergeMap`
- **Filtering**: `filter`, `debounceTime`, `distinctUntilChanged`
- **Combination**: `combineLatest`, `forkJoin`, `withLatestFrom`
- **Error Handling**: `catchError`, `retry`
- **Utility**: `tap`, `shareReplay`, `take`, `takeUntil`

**Key Takeaway:** Choose the right operator for the job. `switchMap` for search, `forkJoin` for parallel loads, `combineLatest` for combining streams.
