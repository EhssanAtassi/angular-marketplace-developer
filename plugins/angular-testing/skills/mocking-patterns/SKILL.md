# Angular Mocking Patterns

Comprehensive guide to mocking patterns, test doubles, spies, and HTTP testing in Angular applications.

## Table of Contents

1. [Test Doubles Overview](#test-doubles-overview)
2. [Jasmine Spies](#jasmine-spies)
3. [Service Mocking](#service-mocking)
4. [HTTP Mocking](#http-mocking)
5. [Observable Mocking](#observable-mocking)
6. [Component Mocking](#component-mocking)
7. [Router & Location Mocking](#router--location-mocking)
8. [Form Mocking](#form-mocking)
9. [LocalStorage & SessionStorage](#localstorage--sessionstorage)
10. [Advanced Patterns](#advanced-patterns)

---

## Test Doubles Overview

### Types of Test Doubles

```typescript
// 1. DUMMY - Passed but never used
class DummyLogger implements Logger {
  log(message: string): void {
    // Does nothing
  }
}

// 2. STUB - Returns predefined responses
class StubUserService {
  getUser(id: number): Observable<User> {
    return of({ id, name: 'Test User', email: 'test@example.com' });
  }
}

// 3. SPY - Records interactions
const spy = jasmine.createSpy('getData');
spy.and.returnValue('mock data');

// 4. MOCK - Pre-programmed with expectations
const mock = jasmine.createSpyObj('UserService', ['getUser', 'updateUser']);
mock.getUser.and.returnValue(of({ id: 1, name: 'John' }));

// 5. FAKE - Working simplified implementation
class FakeAuthService {
  private authenticated = false;

  login(): Observable<boolean> {
    this.authenticated = true;
    return of(true);
  }

  isAuthenticated(): boolean {
    return this.authenticated;
  }
}
```

### When to Use Each

| Type | Use When | Example |
|------|----------|---------|
| Dummy | Need to pass parameters | Logger that's never called |
| Stub | Need consistent responses | API returning test data |
| Spy | Need to verify calls | Track method invocations |
| Mock | Need behavior verification | Service with expectations |
| Fake | Need working alternative | In-memory database |

---

## Jasmine Spies

### Creating Spies

```typescript
// Method 1: Standalone spy
const spy = jasmine.createSpy('myFunction');

// Method 2: Spy on existing object
const service = new UserService();
spyOn(service, 'getData');

// Method 3: Spy object (multiple methods)
const spyObj = jasmine.createSpyObj('UserService', [
  'getUser',
  'updateUser',
  'deleteUser'
]);

// Method 4: Spy object with properties
const spyObjWithProps = jasmine.createSpyObj('UserService', 
  ['getUser'], // methods
  { currentUser: { id: 1, name: 'John' } } // properties
);
```

### Spy Return Values

```typescript
// Return single value
spy.and.returnValue('mock value');

// Return multiple values in sequence
spy.and.returnValues('first', 'second', 'third');

// Return Observable
spy.and.returnValue(of({ id: 1, name: 'John' }));

// Custom implementation
spy.and.callFake((arg) => {
  if (arg === 'admin') {
    return of({ role: 'admin' });
  }
  return of({ role: 'user' });
});

// Call original method
spy.and.callThrough();

// Throw error
spy.and.throwError('Something went wrong');

// Return promise
spy.and.resolveTo({ id: 1 }); // Promise.resolve
spy.and.rejectWith(new Error('Failed')); // Promise.reject
```

### Spy Assertions

```typescript
// Basic assertions
expect(spy).toHaveBeenCalled();
expect(spy).toHaveBeenCalledTimes(2);
expect(spy).not.toHaveBeenCalled();

// Argument assertions
expect(spy).toHaveBeenCalledWith('arg1', 'arg2');
expect(spy).toHaveBeenCalledWith(jasmine.any(String));
expect(spy).toHaveBeenCalledWith(jasmine.objectContaining({ id: 1 }));

// Order assertions
expect(spy1).toHaveBeenCalledBefore(spy2);

// Most recent call
expect(spy).toHaveBeenCalledWith('last call args');

// Reset spy
spy.calls.reset();
```

### Spy Properties

```typescript
const spy = jasmine.createSpy('myFunction');
spy('arg1', 'arg2');

// Access call information
spy.calls.count();           // 1
spy.calls.argsFor(0);        // ['arg1', 'arg2']
spy.calls.allArgs();         // [['arg1', 'arg2']]
spy.calls.all();             // Full call details
spy.calls.mostRecent();      // Last call details
spy.calls.first();           // First call details
```

---

## Service Mocking

### Simple Service Mock

```typescript
describe('UserComponent', () => {
  let component: UserComponent;
  let userService: jasmine.SpyObj<UserService>;

  beforeEach(() => {
    // Create spy object with all methods
    const spy = jasmine.createSpyObj('UserService', [
      'getUser',
      'getUsers',
      'updateUser',
      'deleteUser'
    ]);

    TestBed.configureTestingModule({
      declarations: [UserComponent],
      providers: [
        { provide: UserService, useValue: spy }
      ]
    });

    userService = TestBed.inject(UserService) as jasmine.SpyObj<UserService>;
    component = new UserComponent(userService);
  });

  it('should load user on init', () => {
    const mockUser = { id: 1, name: 'John Doe' };
    userService.getUser.and.returnValue(of(mockUser));

    component.ngOnInit();

    expect(userService.getUser).toHaveBeenCalledWith(1);
    expect(component.user).toEqual(mockUser);
  });
});
```

### Service with Properties

```typescript
describe('AuthComponent', () => {
  let authService: jasmine.SpyObj<AuthService>;

  beforeEach(() => {
    authService = jasmine.createSpyObj(
      'AuthService',
      ['login', 'logout'],
      {
        currentUser: of({ id: 1, name: 'John' }),
        isAuthenticated: true
      }
    );
  });

  it('should display current user', () => {
    expect(authService.isAuthenticated).toBe(true);
    
    authService.currentUser.subscribe(user => {
      expect(user.name).toBe('John');
    });
  });
});
```

### Mocking Service Dependencies

```typescript
describe('ProductService', () => {
  let service: ProductService;
  let http: jasmine.SpyObj<HttpClient>;
  let cache: jasmine.SpyObj<CacheService>;
  let logger: jasmine.SpyObj<LoggerService>;

  beforeEach(() => {
    const httpSpy = jasmine.createSpyObj('HttpClient', ['get', 'post', 'put', 'delete']);
    const cacheSpy = jasmine.createSpyObj('CacheService', ['get', 'set', 'clear']);
    const loggerSpy = jasmine.createSpyObj('LoggerService', ['log', 'error']);

    TestBed.configureTestingModule({
      providers: [
        ProductService,
        { provide: HttpClient, useValue: httpSpy },
        { provide: CacheService, useValue: cacheSpy },
        { provide: LoggerService, useValue: loggerSpy }
      ]
    });

    service = TestBed.inject(ProductService);
    http = TestBed.inject(HttpClient) as jasmine.SpyObj<HttpClient>;
    cache = TestBed.inject(CacheService) as jasmine.SpyObj<CacheService>;
    logger = TestBed.inject(LoggerService) as jasmine.SpyObj<LoggerService>;
  });

  it('should use cached data when available', () => {
    const cachedProduct = { id: 1, name: 'Cached Product' };
    cache.get.and.returnValue(cachedProduct);

    service.getProduct(1).subscribe(product => {
      expect(product).toEqual(cachedProduct);
      expect(http.get).not.toHaveBeenCalled();
      expect(logger.log).toHaveBeenCalledWith('Using cached product');
    });
  });
});
```

---

## HTTP Mocking

### HttpClientTestingModule Setup

```typescript
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';

describe('ApiService', () => {
  let service: ApiService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [ApiService]
    });

    service = TestBed.inject(ApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify(); // Verify no outstanding requests
  });
});
```

### GET Request Mocking

```typescript
it('should fetch users', () => {
  const mockUsers = [
    { id: 1, name: 'John' },
    { id: 2, name: 'Jane' }
  ];

  service.getUsers().subscribe(users => {
    expect(users).toEqual(mockUsers);
    expect(users.length).toBe(2);
  });

  const req = httpMock.expectOne('/api/users');
  expect(req.request.method).toBe('GET');
  req.flush(mockUsers);
});

it('should handle query parameters', () => {
  service.getUsers({ role: 'admin', active: true }).subscribe();

  const req = httpMock.expectOne(req => 
    req.url === '/api/users' &&
    req.params.get('role') === 'admin' &&
    req.params.get('active') === 'true'
  );
  req.flush([]);
});
```

### POST/PUT/DELETE Mocking

```typescript
it('should create user', () => {
  const newUser = { name: 'John', email: 'john@example.com' };
  const createdUser = { id: 1, ...newUser };

  service.createUser(newUser).subscribe(user => {
    expect(user).toEqual(createdUser);
  });

  const req = httpMock.expectOne('/api/users');
  expect(req.request.method).toBe('POST');
  expect(req.request.body).toEqual(newUser);
  req.flush(createdUser);
});

it('should update user', () => {
  const updatedUser = { id: 1, name: 'John Updated' };

  service.updateUser(1, updatedUser).subscribe();

  const req = httpMock.expectOne('/api/users/1');
  expect(req.request.method).toBe('PUT');
  req.flush(updatedUser);
});

it('should delete user', () => {
  service.deleteUser(1).subscribe();

  const req = httpMock.expectOne('/api/users/1');
  expect(req.request.method).toBe('DELETE');
  req.flush(null);
});
```

### HTTP Headers Mocking

```typescript
it('should send authorization header', () => {
  service.getProtectedData().subscribe();

  const req = httpMock.expectOne('/api/protected');
  expect(req.request.headers.get('Authorization')).toBe('Bearer fake-token');
  req.flush({ data: 'secret' });
});

it('should set custom headers', () => {
  service.uploadFile(new FormData()).subscribe();

  const req = httpMock.expectOne('/api/upload');
  expect(req.request.headers.get('Content-Type')).toContain('multipart/form-data');
  req.flush({ success: true });
});
```

### HTTP Error Handling

```typescript
it('should handle 404 error', () => {
  service.getUser(999).subscribe({
    next: () => fail('should have failed'),
    error: (error) => {
      expect(error.status).toBe(404);
      expect(error.statusText).toBe('Not Found');
    }
  });

  const req = httpMock.expectOne('/api/users/999');
  req.flush('User not found', { 
    status: 404, 
    statusText: 'Not Found' 
  });
});

it('should handle 500 error with message', () => {
  service.updateUser(1, {}).subscribe({
    error: (error) => {
      expect(error.status).toBe(500);
      expect(error.error.message).toBe('Server error');
    }
  });

  const req = httpMock.expectOne('/api/users/1');
  req.flush(
    { message: 'Server error' },
    { status: 500, statusText: 'Internal Server Error' }
  );
});

it('should handle network error', () => {
  service.getUsers().subscribe({
    error: (error) => {
      expect(error.error.type).toBe('error');
    }
  });

  const req = httpMock.expectOne('/api/users');
  req.error(new ErrorEvent('Network error'));
});
```

### Multiple HTTP Requests

```typescript
it('should handle multiple simultaneous requests', () => {
  service.getUser(1).subscribe();
  service.getUser(2).subscribe();
  service.getUser(3).subscribe();

  const requests = httpMock.match(req => req.url.startsWith('/api/users/'));
  expect(requests.length).toBe(3);

  requests[0].flush({ id: 1, name: 'User 1' });
  requests[1].flush({ id: 2, name: 'User 2' });
  requests[2].flush({ id: 3, name: 'User 3' });
});

it('should handle sequential dependent requests', fakeAsync(() => {
  let userData: any;
  let postsData: any;

  // First request - get user
  service.getUser(1).subscribe(user => {
    userData = user;
  });

  let req = httpMock.expectOne('/api/users/1');
  req.flush({ id: 1, name: 'John' });
  tick();

  // Second request - get user's posts (depends on first)
  service.getUserPosts(userData.id).subscribe(posts => {
    postsData = posts;
  });

  req = httpMock.expectOne('/api/users/1/posts');
  req.flush([{ id: 1, title: 'Post 1' }]);
  tick();

  expect(userData).toBeDefined();
  expect(postsData.length).toBe(1);
}));
```

---

## Observable Mocking

### Basic Observable Mocking

```typescript
import { of, throwError, EMPTY, NEVER } from 'rxjs';
import { delay } from 'rxjs/operators';

// Success response
service.getData.and.returnValue(of({ data: 'test' }));

// Error response
service.getData.and.returnValue(
  throwError(() => new Error('Failed'))
);

// Empty observable
service.getData.and.returnValue(EMPTY);

// Never completing observable
service.getData.and.returnValue(NEVER);

// Delayed response
service.getData.and.returnValue(
  of({ data: 'test' }).pipe(delay(1000))
);
```

### Subject Mocking

```typescript
import { Subject, BehaviorSubject, ReplaySubject } from 'rxjs';

describe('NotificationService', () => {
  let service: NotificationService;
  let notificationSubject: Subject<string>;

  beforeEach(() => {
    notificationSubject = new Subject<string>();
    
    const spy = jasmine.createSpyObj('NotificationService', 
      ['notify'],
      { notifications$: notificationSubject.asObservable() }
    );

    service = spy;
  });

  it('should receive notifications', (done) => {
    const messages: string[] = [];

    service.notifications$.subscribe(msg => {
      messages.push(msg);
      
      if (messages.length === 2) {
        expect(messages).toEqual(['Hello', 'World']);
        done();
      }
    });

    notificationSubject.next('Hello');
    notificationSubject.next('World');
  });
});

// BehaviorSubject with initial value
it('should have initial value', () => {
  const subject = new BehaviorSubject<number>(0);
  const spy = jasmine.createSpyObj('CounterService', 
    ['increment'],
    { count$: subject.asObservable() }
  );

  spy.count$.subscribe(count => {
    expect(count).toBe(0);
  });

  subject.next(1);
  
  spy.count$.subscribe(count => {
    expect(count).toBe(1);
  });
});
```

### Async Observable Testing

```typescript
it('should handle async data', fakeAsync(() => {
  let result: any;

  service.getData().pipe(
    delay(1000)
  ).subscribe(data => {
    result = data;
  });

  expect(result).toBeUndefined();

  tick(1000); // Advance time

  expect(result).toEqual({ data: 'test' });
}));

it('should handle debounced input', fakeAsync(() => {
  const spy = jasmine.createSpy('callback');

  component.searchInput.pipe(
    debounceTime(300)
  ).subscribe(spy);

  component.searchInput.next('a');
  tick(100);
  component.searchInput.next('ab');
  tick(100);
  component.searchInput.next('abc');
  tick(300);

  expect(spy).toHaveBeenCalledTimes(1);
  expect(spy).toHaveBeenCalledWith('abc');
}));
```

---

## Component Mocking

### Mock Child Components

```typescript
// Create mock component
@Component({
  selector: 'app-child',
  template: '<div>Mock Child</div>'
})
class MockChildComponent {
  @Input() data: any;
  @Output() action = new EventEmitter();
}

describe('ParentComponent', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [
        ParentComponent,
        MockChildComponent // Use mock instead of real
      ]
    }).compileComponents();
  });

  it('should pass data to child', () => {
    const fixture = TestBed.createComponent(ParentComponent);
    const component = fixture.componentInstance;
    const childDebugElement = fixture.debugElement.query(
      By.directive(MockChildComponent)
    );
    const childComponent = childDebugElement.componentInstance as MockChildComponent;

    component.parentData = { value: 'test' };
    fixture.detectChanges();

    expect(childComponent.data).toEqual({ value: 'test' });
  });

  it('should handle child events', () => {
    const fixture = TestBed.createComponent(ParentComponent);
    const component = fixture.componentInstance;
    const childDebugElement = fixture.debugElement.query(
      By.directive(MockChildComponent)
    );
    const childComponent = childDebugElement.componentInstance as MockChildComponent;

    spyOn(component, 'handleAction');

    childComponent.action.emit('test-data');

    expect(component.handleAction).toHaveBeenCalledWith('test-data');
  });
});
```

### NO_ERRORS_SCHEMA

```typescript
// Alternative: Use NO_ERRORS_SCHEMA to ignore child components
import { NO_ERRORS_SCHEMA } from '@angular/core';

describe('ParentComponent', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [ParentComponent],
      schemas: [NO_ERRORS_SCHEMA] // Ignore unknown elements
    }).compileComponents();
  });

  it('should work without child component implementation', () => {
    const fixture = TestBed.createComponent(ParentComponent);
    expect(fixture.componentInstance).toBeTruthy();
  });
});
```

---

## Router & Location Mocking

### Router Mocking

```typescript
import { Router } from '@angular/router';

describe('NavigationComponent', () => {
  let component: NavigationComponent;
  let router: jasmine.SpyObj<Router>;

  beforeEach(() => {
    const routerSpy = jasmine.createSpyObj('Router', [
      'navigate',
      'navigateByUrl'
    ]);

    TestBed.configureTestingModule({
      declarations: [NavigationComponent],
      providers: [
        { provide: Router, useValue: routerSpy }
      ]
    });

    router = TestBed.inject(Router) as jasmine.SpyObj<Router>;
    component = new NavigationComponent(router);
  });

  it('should navigate to dashboard', () => {
    component.goToDashboard();

    expect(router.navigate).toHaveBeenCalledWith(['/dashboard']);
  });

  it('should navigate with query params', () => {
    component.goToProducts({ category: 'electronics' });

    expect(router.navigate).toHaveBeenCalledWith(
      ['/products'],
      { queryParams: { category: 'electronics' } }
    );
  });
});
```

### ActivatedRoute Mocking

```typescript
import { ActivatedRoute } from '@angular/router';
import { of } from 'rxjs';

describe('ProductDetailComponent', () => {
  let component: ProductDetailComponent;
  let route: ActivatedRoute;

  beforeEach(() => {
    const routeMock = {
      snapshot: {
        paramMap: {
          get: (key: string) => '123'
        }
      },
      params: of({ id: '123' }),
      queryParams: of({ tab: 'details' }),
      data: of({ product: { id: 1, name: 'Test' } })
    };

    TestBed.configureTestingModule({
      declarations: [ProductDetailComponent],
      providers: [
        { provide: ActivatedRoute, useValue: routeMock }
      ]
    });

    route = TestBed.inject(ActivatedRoute);
    component = new ProductDetailComponent(route);
  });

  it('should read route params', () => {
    expect(component.productId).toBe('123');
  });

  it('should subscribe to query params', () => {
    component.ngOnInit();
    expect(component.activeTab).toBe('details');
  });
});
```

### RouterTestingModule

```typescript
import { RouterTestingModule } from '@angular/router/testing';
import { Location } from '@angular/common';
import { Router } from '@angular/router';

describe('Navigation Integration', () => {
  let router: Router;
  let location: Location;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        RouterTestingModule.withRoutes([
          { path: 'home', component: HomeComponent },
          { path: 'about', component: AboutComponent }
        ])
      ],
      declarations: [HomeComponent, AboutComponent]
    }).compileComponents();

    router = TestBed.inject(Router);
    location = TestBed.inject(Location);
  });

  it('should navigate to home', fakeAsync(() => {
    router.navigate(['/home']);
    tick();

    expect(location.path()).toBe('/home');
  }));
});
```

---

## Form Mocking

### FormBuilder & FormControl Mocking

```typescript
import { FormBuilder, FormGroup, Validators } from '@angular/forms';

describe('UserFormComponent', () => {
  let component: UserFormComponent;
  let fb: FormBuilder;

  beforeEach(() => {
    fb = new FormBuilder();
    component = new UserFormComponent(fb);
    component.ngOnInit();
  });

  it('should create form with controls', () => {
    expect(component.userForm.get('name')).toBeDefined();
    expect(component.userForm.get('email')).toBeDefined();
  });

  it('should validate required fields', () => {
    const name = component.userForm.get('name');
    expect(name?.valid).toBeFalsy();
    expect(name?.hasError('required')).toBeTruthy();

    name?.setValue('John Doe');
    expect(name?.valid).toBeTruthy();
  });

  it('should validate email format', () => {
    const email = component.userForm.get('email');
    
    email?.setValue('invalid-email');
    expect(email?.hasError('email')).toBeTruthy();

    email?.setValue('valid@example.com');
    expect(email?.valid).toBeTruthy();
  });
});
```

### Custom Validators Mocking

```typescript
// Custom validator
export function ageValidator(min: number, max: number): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const age = control.value;
    if (age < min || age > max) {
      return { ageRange: { min, max, actual: age } };
    }
    return null;
  };
}

// Testing custom validator
describe('Age Validator', () => {
  it('should validate age range', () => {
    const control = new FormControl(25);
    const validator = ageValidator(18, 65);

    expect(validator(control)).toBeNull(); // Valid

    control.setValue(10);
    expect(validator(control)).toEqual({
      ageRange: { min: 18, max: 65, actual: 10 }
    });

    control.setValue(70);
    expect(validator(control)).toEqual({
      ageRange: { min: 18, max: 65, actual: 70 }
    });
  });
});
```

---

## LocalStorage & SessionStorage

### Storage Mocking

```typescript
describe('StorageService', () => {
  let service: StorageService;
  let store: { [key: string]: string };

  beforeEach(() => {
    store = {};

    spyOn(localStorage, 'getItem').and.callFake((key: string) => {
      return store[key] || null;
    });

    spyOn(localStorage, 'setItem').and.callFake((key: string, value: string) => {
      store[key] = value;
    });

    spyOn(localStorage, 'removeItem').and.callFake((key: string) => {
      delete store[key];
    });

    spyOn(localStorage, 'clear').and.callFake(() => {
      store = {};
    });

    service = new StorageService();
  });

  it('should save data to localStorage', () => {
    service.setItem('key', 'value');
    
    expect(localStorage.setItem).toHaveBeenCalledWith('key', 'value');
    expect(store['key']).toBe('value');
  });

  it('should retrieve data from localStorage', () => {
    store['key'] = 'value';
    
    const result = service.getItem('key');
    
    expect(localStorage.getItem).toHaveBeenCalledWith('key');
    expect(result).toBe('value');
  });

  it('should remove item from localStorage', () => {
    store['key'] = 'value';
    
    service.removeItem('key');
    
    expect(localStorage.removeItem).toHaveBeenCalledWith('key');
    expect(store['key']).toBeUndefined();
  });
});
```

---

## Advanced Patterns

### Partial Mocking

```typescript
// Mock only specific methods, keep others real
describe('ComplexService', () => {
  let service: ComplexService;

  beforeEach(() => {
    service = new ComplexService();
    spyOn(service, 'expensiveOperation').and.returnValue('mocked');
    // Other methods use real implementation
  });

  it('should use mocked method', () => {
    const result = service.doSomething();
    expect(service.expensiveOperation).toHaveBeenCalled();
  });
});
```

### Spy Chaining

```typescript
it('should chain multiple service calls', () => {
  const userService = jasmine.createSpyObj('UserService', ['getUser']);
  const orderService = jasmine.createSpyObj('OrderService', ['getOrders']);

  userService.getUser.and.returnValue(of({ id: 1, name: 'John' }));
  orderService.getOrders.and.returnValue(of([{ id: 1, total: 100 }]));

  component.loadUserData(1);

  expect(userService.getUser).toHaveBeenCalledWith(1);
  expect(orderService.getOrders).toHaveBeenCalledWith(1);
});
```

### Conditional Mocking

```typescript
it('should mock based on arguments', () => {
  const service = jasmine.createSpyObj('ApiService', ['getData']);

  service.getData.and.callFake((id: number) => {
    if (id === 1) {
      return of({ data: 'admin' });
    } else if (id === 2) {
      return of({ data: 'user' });
    } else {
      return throwError(() => new Error('Not found'));
    }
  });

  service.getData(1).subscribe(result => {
    expect(result.data).toBe('admin');
  });

  service.getData(2).subscribe(result => {
    expect(result.data).toBe('user');
  });

  service.getData(999).subscribe({
    error: (error) => {
      expect(error.message).toBe('Not found');
    }
  });
});
```

### Dynamic Mock Configuration

```typescript
describe('ConfigurableService', () => {
  let service: MyService;
  let apiService: jasmine.SpyObj<ApiService>;

  function configureTest(config: {
    shouldSucceed: boolean;
    delay?: number;
    returnValue?: any;
  }) {
    if (config.shouldSucceed) {
      let obs = of(config.returnValue || { success: true });
      if (config.delay) {
        obs = obs.pipe(delay(config.delay));
      }
      apiService.getData.and.returnValue(obs);
    } else {
      apiService.getData.and.returnValue(
        throwError(() => new Error('Failed'))
      );
    }
  }

  beforeEach(() => {
    apiService = jasmine.createSpyObj('ApiService', ['getData']);
    service = new MyService(apiService);
  });

  it('should handle success case', () => {
    configureTest({ 
      shouldSucceed: true,
      returnValue: { data: 'test' }
    });

    service.loadData().subscribe(result => {
      expect(result.data).toBe('test');
    });
  });

  it('should handle error case', () => {
    configureTest({ shouldSucceed: false });

    service.loadData().subscribe({
      error: (error) => {
        expect(error.message).toBe('Failed');
      }
    });
  });
});
```

---

## Best Practices

### ✅ DO

```typescript
// Use spy objects for dependencies
const spy = jasmine.createSpyObj('Service', ['method']);

// Reset spies between tests
afterEach(() => {
  spy.calls.reset();
});

// Use meaningful test data
const mockUser = { id: 1, name: 'John Doe', email: 'john@example.com' };

// Mock at the right level
// Component tests: Mock services
// Service tests: Mock HTTP
// E2E tests: Mock nothing

// Verify spy calls explicitly
expect(spy.method).toHaveBeenCalledWith('expected', 'arguments');
```

### ❌ DON'T

```typescript
// Don't use real services in unit tests
TestBed.configureTestingModule({
  providers: [RealDatabaseService] // ❌ Bad
});

// Don't over-mock
spyOn(Math, 'random');  // ❌ Don't mock language features
spyOn(Array.prototype, 'push'); // ❌ Don't mock prototypes

// Don't create brittle mocks
spy.and.returnValue({ /* huge object */ }); // ❌ Too specific

// Don't forget to verify HTTP calls
// Missing httpMock.verify() in afterEach ❌
```

---

## Quick Reference

### Common Spy Patterns

```typescript
// Return value
spy.and.returnValue(value);

// Return Observable
spy.and.returnValue(of(value));

// Return Promise
spy.and.resolveTo(value);

// Throw error
spy.and.throwError('error');

// Custom logic
spy.and.callFake((arg) => { /* logic */ });

// Call real method
spy.and.callThrough();
```

### Common Assertions

```typescript
expect(spy).toHaveBeenCalled();
expect(spy).toHaveBeenCalledTimes(n);
expect(spy).toHaveBeenCalledWith(arg1, arg2);
expect(spy).not.toHaveBeenCalled();
```

### HTTP Testing

```typescript
const req = httpMock.expectOne(url);
expect(req.request.method).toBe('GET');
req.flush(mockData);
httpMock.verify(); // In afterEach
```

---

*Master mocking for clean, maintainable tests! 🎭*
