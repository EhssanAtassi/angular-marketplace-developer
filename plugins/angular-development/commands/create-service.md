---
name: create-service
description: Generate Angular service with HTTP methods, error handling, caching, and proper dependency injection
---

Generate a production-ready Angular service with HTTP communication, state management, and error handling.

## Service Types

Ask the user which type:

1. **Data Service** - HTTP API communication
2. **State Service** - Global state management
3. **Utility Service** - Helper functions and utilities
4. **Facade Service** - Simplifies complex subsystems

## Information Needed

1. **Service name** - kebab-case (e.g., `user-service`)
2. **Service type** - Data, State, Utility, or Facade
3. **API endpoint** - Base URL for data services
4. **Data model** - What type of data does it handle?
5. **Operations needed** - CRUD? Search? Filter?
6. **Caching?** - Should responses be cached?

## Service Structure

### Data Service Template

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry, shareReplay } from 'rxjs/operators';
import { environment } from '@environments/environment';

export interface DataModel {
  id: string;
  // Add properties
}

export interface QueryParams {
  page?: number;
  limit?: number;
  search?: string;
}

@Injectable({
  providedIn: 'root'  // Singleton service
})
export class DataService {
  private http = inject(HttpClient);
  private baseUrl = `${environment.apiUrl}/data`;
  
  // GET all with pagination
  getAll(params?: QueryParams): Observable<DataModel[]> {
    let httpParams = new HttpParams();
    
    if (params) {
      Object.keys(params).forEach(key => {
        if (params[key as keyof QueryParams] !== undefined) {
          httpParams = httpParams.set(key, params[key as keyof QueryParams]!.toString());
        }
      });
    }
    
    return this.http.get<DataModel[]>(this.baseUrl, { params: httpParams }).pipe(
      retry(2),
      catchError(this.handleError)
    );
  }
  
  // GET by ID with caching
  getById(id: string): Observable<DataModel> {
    return this.http.get<DataModel>(`${this.baseUrl}/${id}`).pipe(
      shareReplay(1),  // Cache for multiple subscribers
      catchError(this.handleError)
    );
  }
  
  // POST - Create
  create(data: Omit<DataModel, 'id'>): Observable<DataModel> {
    return this.http.post<DataModel>(this.baseUrl, data).pipe(
      catchError(this.handleError)
    );
  }
  
  // PUT - Update
  update(id: string, data: Partial<DataModel>): Observable<DataModel> {
    return this.http.put<DataModel>(`${this.baseUrl}/${id}`, data).pipe(
      catchError(this.handleError)
    );
  }
  
  // PATCH - Partial update
  patch(id: string, data: Partial<DataModel>): Observable<DataModel> {
    return this.http.patch<DataModel>(`${this.baseUrl}/${id}`, data).pipe(
      catchError(this.handleError)
    );
  }
  
  // DELETE
  delete(id: string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`).pipe(
      catchError(this.handleError)
    );
  }
  
  // Search
  search(query: string): Observable<DataModel[]> {
    return this.http.get<DataModel[]>(`${this.baseUrl}/search`, {
      params: { q: query }
    }).pipe(
      catchError(this.handleError)
    );
  }
  
  // Error handling
  private handleError(error: any): Observable<never> {
    console.error('Service error:', error);
    
    let errorMessage = 'An error occurred';
    
    if (error.error instanceof ErrorEvent) {
      // Client-side error
      errorMessage = `Error: ${error.error.message}`;
    } else {
      // Server-side error
      errorMessage = `Error Code: ${error.status}\nMessage: ${error.message}`;
    }
    
    return throwError(() => new Error(errorMessage));
  }
}
```

### State Service Template

```typescript
import { Injectable, signal, computed } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

export interface AppState {
  loading: boolean;
  data: DataModel[];
  selectedId: string | null;
  error: string | null;
}

@Injectable({
  providedIn: 'root'
})
export class StateService {
  // Using Signals (Modern approach)
  private dataSignal = signal<DataModel[]>([]);
  private loadingSignal = signal(false);
  private errorSignal = signal<string | null>(null);
  
  // Public readonly signals
  readonly data = this.dataSignal.asReadonly();
  readonly loading = this.loadingSignal.asReadonly();
  readonly error = this.errorSignal.asReadonly();
  
  // Computed signals
  readonly itemCount = computed(() => this.data().length);
  readonly hasData = computed(() => this.data().length > 0);
  
  // OR using BehaviorSubject (Traditional approach)
  private stateSubject = new BehaviorSubject<AppState>({
    loading: false,
    data: [],
    selectedId: null,
    error: null
  });
  
  state$ = this.stateSubject.asObservable();
  
  // Setters
  setData(data: DataModel[]) {
    this.dataSignal.set(data);
  }
  
  addItem(item: DataModel) {
    this.dataSignal.update(current => [...current, item]);
  }
  
  updateItem(id: string, updates: Partial<DataModel>) {
    this.dataSignal.update(current =>
      current.map(item => item.id === id ? { ...item, ...updates } : item)
    );
  }
  
  removeItem(id: string) {
    this.dataSignal.update(current => current.filter(item => item.id !== id));
  }
  
  setLoading(loading: boolean) {
    this.loadingSignal.set(loading);
  }
  
  setError(error: string | null) {
    this.errorSignal.set(error);
  }
  
  clearState() {
    this.dataSignal.set([]);
    this.errorSignal.set(null);
    this.loadingSignal.set(false);
  }
}
```

### Facade Service Template

```typescript
import { Injectable, inject } from '@angular/core';
import { Observable, forkJoin, combineLatest } from 'rxjs';
import { map, switchMap, tap } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class FacadeService {
  private dataService = inject(DataService);
  private stateService = inject(StateService);
  private cacheService = inject(CacheService);
  
  // Simplified API for components
  readonly data$ = this.stateService.data;
  readonly loading$ = this.stateService.loading;
  
  // Complex operation simplified
  loadData(): Observable<DataModel[]> {
    this.stateService.setLoading(true);
    
    return this.dataService.getAll().pipe(
      tap(data => {
        this.stateService.setData(data);
        this.cacheService.set('data', data);
        this.stateService.setLoading(false);
      }),
      catchError(error => {
        this.stateService.setError(error.message);
        this.stateService.setLoading(false);
        return throwError(() => error);
      })
    );
  }
  
  // Orchestrate multiple services
  initialize(): Observable<any> {
    return forkJoin({
      config: this.dataService.getConfig(),
      user: this.dataService.getUser(),
      permissions: this.dataService.getPermissions()
    }).pipe(
      tap(({ config, user, permissions }) => {
        this.stateService.setConfig(config);
        this.stateService.setUser(user);
        this.stateService.setPermissions(permissions);
      })
    );
  }
}
```

### Utility Service Template

```typescript
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class UtilityService {
  // Date utilities
  formatDate(date: Date | string): string {
    return new Date(date).toLocaleDateString('en-US');
  }
  
  // String utilities
  capitalize(str: string): string {
    return str.charAt(0).toUpperCase() + str.slice(1);
  }
  
  slugify(str: string): string {
    return str
      .toLowerCase()
      .replace(/[^\w\s-]/g, '')
      .replace(/[\s_-]+/g, '-')
      .replace(/^-+|-+$/g, '');
  }
  
  // Array utilities
  groupBy<T>(array: T[], key: keyof T): Record<string, T[]> {
    return array.reduce((result, item) => {
      const groupKey = String(item[key]);
      if (!result[groupKey]) {
        result[groupKey] = [];
      }
      result[groupKey].push(item);
      return result;
    }, {} as Record<string, T[]>);
  }
  
  // Number utilities
  formatCurrency(amount: number, currency = 'USD'): string {
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency
    }).format(amount);
  }
  
  // Validation utilities
  isValidEmail(email: string): boolean {
    const regex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;
    return regex.test(email);
  }
}
```

## Service with Caching

```typescript
import { Injectable, inject } from '@angular/core';
import { Observable, of, timer } from 'rxjs';
import { tap, switchMap, shareReplay } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class CachedDataService {
  private http = inject(HttpClient);
  private cache = new Map<string, { data: any; timestamp: number }>();
  private TTL = 5 * 60 * 1000; // 5 minutes
  
  getData(key: string, forceRefresh = false): Observable<DataModel[]> {
    const cached = this.cache.get(key);
    
    // Return cached if valid
    if (!forceRefresh && cached && Date.now() - cached.timestamp < this.TTL) {
      return of(cached.data);
    }
    
    // Fetch fresh data
    return this.http.get<DataModel[]>(`/api/${key}`).pipe(
      tap(data => {
        this.cache.set(key, {
          data,
          timestamp: Date.now()
        });
      }),
      shareReplay(1)
    );
  }
  
  invalidateCache(key?: string) {
    if (key) {
      this.cache.delete(key);
    } else {
      this.cache.clear();
    }
  }
}
```

## Test Template

```typescript
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { DataService } from './data.service';

describe('DataService', () => {
  let service: DataService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [DataService]
    });
    
    service = TestBed.inject(DataService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should fetch data', () => {
    const mockData = [{ id: '1', name: 'Test' }];
    
    service.getAll().subscribe(data => {
      expect(data).toEqual(mockData);
    });
    
    const req = httpMock.expectOne(`${service['baseUrl']}`);
    expect(req.request.method).toBe('GET');
    req.flush(mockData);
  });
});
```

## Best Practices

1. **Use `providedIn: 'root'`** for singleton services
2. **Inject dependencies with `inject()`** function
3. **Handle errors properly** with catchError
4. **Add retry logic** for network requests
5. **Cache responses** when appropriate
6. **Use interfaces** for data models
7. **Add JSDoc comments** for complex methods
8. **Write tests** for all public methods
9. **Use environment variables** for URLs
10. **Use HttpParams** for query parameters

## Usage

```bash
/angular-development:create-service

# Natural language
"Create a data service for products with CRUD operations"
"Generate a state service for shopping cart"
```
