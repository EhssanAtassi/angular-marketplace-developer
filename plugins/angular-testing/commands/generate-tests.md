# Generate Tests Command

Generate comprehensive unit and integration tests for Angular components, services, and modules.

## Command Usage

```bash
/angular-testing:generate-tests [component|service|module|pipe|directive] <name>
```

## Natural Language Examples

- "Generate tests for my user profile component"
- "Create unit tests for the authentication service"
- "Write tests for the admin guard"
- "Generate tests for the currency pipe"

## What This Command Does

1. **Analyzes the target file** - Understands component/service structure
2. **Creates comprehensive test suite** - Following AAA pattern
3. **Includes all test scenarios** - Happy path, edge cases, errors
4. **Mocks dependencies** - Proper service and HTTP mocking
5. **Adds helpful comments** - Explains complex test logic
6. **Follows best practices** - TestBed configuration, spy objects

## Test Generation Patterns

### Component Tests Include

```typescript
describe('ComponentName', () => {
  // ✅ Component creation and initialization
  // ✅ Input property testing
  // ✅ Output event testing
  // ✅ Method behavior testing
  // ✅ Template rendering
  // ✅ User interaction
  // ✅ Async operations
  // ✅ Error handling
});
```

### Service Tests Include

```typescript
describe('ServiceName', () => {
  // ✅ Service creation
  // ✅ Method functionality
  // ✅ HTTP requests (mocked)
  // ✅ Observable streams
  // ✅ Error handling
  // ✅ State management
  // ✅ Cache behavior
});
```

## Example Output

**Input**: Generate tests for LoginComponent

**Output**:
```typescript
// login.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { ReactiveFormsModule } from '@angular/forms';
import { LoginComponent } from './login.component';
import { AuthService } from '../services/auth.service';
import { Router } from '@angular/router';
import { of, throwError } from 'rxjs';
import { DebugElement } from '@angular/core';
import { By } from '@angular/platform-browser';

describe('LoginComponent', () => {
  let component: LoginComponent;
  let fixture: ComponentFixture<LoginComponent>;
  let authService: jasmine.SpyObj<AuthService>;
  let router: jasmine.SpyObj<Router>;
  let compiled: HTMLElement;

  beforeEach(async () => {
    // Create spy objects for dependencies
    const authServiceSpy = jasmine.createSpyObj('AuthService', ['login', 'logout']);
    const routerSpy = jasmine.createSpyObj('Router', ['navigate']);

    await TestBed.configureTestingModule({
      declarations: [ LoginComponent ],
      imports: [ ReactiveFormsModule ],
      providers: [
        { provide: AuthService, useValue: authServiceSpy },
        { provide: Router, useValue: routerSpy }
      ]
    }).compileComponents();

    authService = TestBed.inject(AuthService) as jasmine.SpyObj<AuthService>;
    router = TestBed.inject(Router) as jasmine.SpyObj<Router>;
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(LoginComponent);
    component = fixture.componentInstance;
    compiled = fixture.nativeElement;
    fixture.detectChanges();
  });

  afterEach(() => {
    fixture.destroy();
  });

  // === COMPONENT CREATION ===
  describe('Component Creation', () => {
    it('should create the component', () => {
      expect(component).toBeTruthy();
    });

    it('should initialize login form with empty values', () => {
      expect(component.loginForm.get('email')?.value).toBe('');
      expect(component.loginForm.get('password')?.value).toBe('');
    });

    it('should have isLoading set to false initially', () => {
      expect(component.isLoading).toBeFalsy();
    });

    it('should have errorMessage set to null initially', () => {
      expect(component.errorMessage).toBeNull();
    });
  });

  // === FORM VALIDATION ===
  describe('Form Validation', () => {
    it('should have required validator on email field', () => {
      const email = component.loginForm.get('email');
      email?.setValue('');
      expect(email?.hasError('required')).toBeTruthy();
    });

    it('should have email validator on email field', () => {
      const email = component.loginForm.get('email');
      email?.setValue('invalid-email');
      expect(email?.hasError('email')).toBeTruthy();
    });

    it('should accept valid email format', () => {
      const email = component.loginForm.get('email');
      email?.setValue('test@example.com');
      expect(email?.valid).toBeTruthy();
    });

    it('should have required validator on password field', () => {
      const password = component.loginForm.get('password');
      password?.setValue('');
      expect(password?.hasError('required')).toBeTruthy();
    });

    it('should have minLength validator on password field', () => {
      const password = component.loginForm.get('password');
      password?.setValue('123');
      expect(password?.hasError('minlength')).toBeTruthy();
    });

    it('should mark form as invalid when fields are empty', () => {
      expect(component.loginForm.valid).toBeFalsy();
    });

    it('should mark form as valid when all fields are correct', () => {
      component.loginForm.setValue({
        email: 'test@example.com',
        password: 'password123'
      });
      expect(component.loginForm.valid).toBeTruthy();
    });
  });

  // === LOGIN FUNCTIONALITY ===
  describe('Login Functionality', () => {
    const validCredentials = {
      email: 'test@example.com',
      password: 'password123'
    };

    beforeEach(() => {
      component.loginForm.setValue(validCredentials);
    });

    it('should not submit when form is invalid', () => {
      component.loginForm.setValue({ email: '', password: '' });
      component.onSubmit();
      expect(authService.login).not.toHaveBeenCalled();
    });

    it('should call authService.login with form values', () => {
      authService.login.and.returnValue(of({ 
        token: 'fake-jwt-token',
        user: { id: 1, email: 'test@example.com' }
      }));

      component.onSubmit();

      expect(authService.login).toHaveBeenCalledWith(
        validCredentials.email,
        validCredentials.password
      );
    });

    it('should set isLoading to true during login', () => {
      authService.login.and.returnValue(of({ token: 'fake-token' }));
      
      component.onSubmit();
      
      expect(component.isLoading).toBeTruthy();
    });

    it('should navigate to dashboard on successful login', (done) => {
      authService.login.and.returnValue(of({ token: 'fake-token' }));
      
      component.onSubmit();
      
      setTimeout(() => {
        expect(router.navigate).toHaveBeenCalledWith(['/dashboard']);
        expect(component.isLoading).toBeFalsy();
        done();
      }, 100);
    });

    it('should display error message on login failure', (done) => {
      const errorResponse = { 
        error: { message: 'Invalid credentials' }
      };
      authService.login.and.returnValue(
        throwError(() => errorResponse)
      );
      
      component.onSubmit();
      
      setTimeout(() => {
        expect(component.errorMessage).toBe('Invalid credentials');
        expect(component.isLoading).toBeFalsy();
        expect(router.navigate).not.toHaveBeenCalled();
        done();
      }, 100);
    });

    it('should clear error message on new submission', () => {
      component.errorMessage = 'Previous error';
      authService.login.and.returnValue(of({ token: 'fake-token' }));
      
      component.onSubmit();
      
      expect(component.errorMessage).toBeNull();
    });
  });

  // === UI INTERACTION ===
  describe('UI Interaction', () => {
    it('should render email input field', () => {
      const emailInput = compiled.querySelector('input[type="email"]');
      expect(emailInput).toBeTruthy();
    });

    it('should render password input field', () => {
      const passwordInput = compiled.querySelector('input[type="password"]');
      expect(passwordInput).toBeTruthy();
    });

    it('should render submit button', () => {
      const submitButton = compiled.querySelector('button[type="submit"]');
      expect(submitButton).toBeTruthy();
    });

    it('should disable submit button when form is invalid', () => {
      const submitButton = compiled.querySelector('button[type="submit"]') as HTMLButtonElement;
      expect(submitButton.disabled).toBeTruthy();
    });

    it('should enable submit button when form is valid', () => {
      component.loginForm.setValue({
        email: 'test@example.com',
        password: 'password123'
      });
      fixture.detectChanges();
      
      const submitButton = compiled.querySelector('button[type="submit"]') as HTMLButtonElement;
      expect(submitButton.disabled).toBeFalsy();
    });

    it('should show loading spinner when isLoading is true', () => {
      component.isLoading = true;
      fixture.detectChanges();
      
      const spinner = compiled.querySelector('.loading-spinner');
      expect(spinner).toBeTruthy();
    });

    it('should display error message when present', () => {
      component.errorMessage = 'Login failed';
      fixture.detectChanges();
      
      const errorElement = compiled.querySelector('.error-message');
      expect(errorElement?.textContent).toContain('Login failed');
    });

    it('should update email value on input', () => {
      const emailInput = compiled.querySelector('input[type="email"]') as HTMLInputElement;
      emailInput.value = 'new@example.com';
      emailInput.dispatchEvent(new Event('input'));
      
      expect(component.loginForm.get('email')?.value).toBe('new@example.com');
    });

    it('should show validation error when email is touched and invalid', () => {
      const email = component.loginForm.get('email');
      email?.markAsTouched();
      email?.setValue('');
      fixture.detectChanges();
      
      const errorElement = compiled.querySelector('.email-error');
      expect(errorElement).toBeTruthy();
    });

    it('should trigger onSubmit when form is submitted', () => {
      spyOn(component, 'onSubmit');
      
      component.loginForm.setValue({
        email: 'test@example.com',
        password: 'password123'
      });
      
      const form = compiled.querySelector('form') as HTMLFormElement;
      form.dispatchEvent(new Event('submit'));
      
      expect(component.onSubmit).toHaveBeenCalled();
    });
  });

  // === EDGE CASES ===
  describe('Edge Cases', () => {
    it('should handle network error gracefully', (done) => {
      component.loginForm.setValue({
        email: 'test@example.com',
        password: 'password123'
      });
      
      authService.login.and.returnValue(
        throwError(() => new Error('Network error'))
      );
      
      component.onSubmit();
      
      setTimeout(() => {
        expect(component.errorMessage).toBe('Network error occurred');
        expect(component.isLoading).toBeFalsy();
        done();
      }, 100);
    });

    it('should prevent multiple simultaneous login requests', () => {
      component.loginForm.setValue({
        email: 'test@example.com',
        password: 'password123'
      });
      
      authService.login.and.returnValue(of({ token: 'fake-token' }));
      
      component.onSubmit();
      component.onSubmit(); // Try to submit again
      
      expect(authService.login).toHaveBeenCalledTimes(1);
    });

    it('should trim whitespace from email', () => {
      component.loginForm.get('email')?.setValue('  test@example.com  ');
      authService.login.and.returnValue(of({ token: 'fake-token' }));
      
      component.onSubmit();
      
      expect(authService.login).toHaveBeenCalledWith(
        'test@example.com',
        jasmine.any(String)
      );
    });
  });
});
```

**Test Coverage Report**:
```
✅ Component Creation (4 tests)
✅ Form Validation (7 tests)
✅ Login Functionality (6 tests)
✅ UI Interaction (10 tests)
✅ Edge Cases (3 tests)
───────────────────────────
Total: 30 tests
Coverage: ~95%
```

## Configuration Tips

### Jasmine/Karma Setup

```typescript
// karma.conf.js
module.exports = function(config) {
  config.set({
    basePath: '',
    frameworks: ['jasmine', '@angular-devkit/build-angular'],
    plugins: [
      require('karma-jasmine'),
      require('karma-chrome-launcher'),
      require('karma-coverage'),
      require('@angular-devkit/build-angular/plugins/karma')
    ],
    coverageReporter: {
      dir: require('path').join(__dirname, './coverage'),
      subdir: '.',
      reporters: [
        { type: 'html' },
        { type: 'text-summary' },
        { type: 'lcovonly' }
      ]
    },
    reporters: ['progress', 'coverage'],
    browsers: ['ChromeHeadless'],
    singleRun: true
  });
};
```

### Jest Setup (Alternative)

```typescript
// jest.config.js
module.exports = {
  preset: 'jest-preset-angular',
  setupFilesAfterEnv: ['<rootDir>/setup-jest.ts'],
  testPathIgnorePatterns: ['/node_modules/', '/dist/'],
  coverageDirectory: 'coverage',
  collectCoverageFrom: [
    'src/app/**/*.ts',
    '!src/app/**/*.spec.ts',
    '!src/app/**/*.module.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
};
```

## Running Tests

```bash
# Run all tests
ng test

# Run with coverage
ng test --code-coverage

# Run specific file
ng test --include='**/login.component.spec.ts'

# Run in headless mode
ng test --browsers=ChromeHeadless --watch=false

# Run with Jest
npm run test:jest
```

## Best Practices Applied

1. **AAA Pattern**: Arrange → Act → Assert structure
2. **Descriptive Names**: Clear "should..." test descriptions
3. **Isolated Tests**: Each test independent and focused
4. **Proper Mocking**: Spy objects for all dependencies
5. **Complete Coverage**: Happy path, errors, edge cases
6. **Cleanup**: afterEach for proper teardown
7. **Async Handling**: Proper async/await and done()

## Usage

```bash
/angular-testing:generate-tests

# Examples
"Generate tests for UserProfileComponent"
"Create tests for DataService"
"Write tests for AuthGuard"
```

## Output Includes

1. **Complete test file** - Ready to run
2. **All imports** - Proper dependencies
3. **TestBed configuration** - Correct module setup
4. **Spy objects** - For all dependencies
5. **Comprehensive scenarios** - All test cases
6. **Comments** - Explaining complex logic
7. **Running instructions** - How to execute tests

---

*Comprehensive testing made easy! 🧪*
