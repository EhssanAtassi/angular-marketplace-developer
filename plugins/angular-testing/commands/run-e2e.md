# Run E2E Tests Command

Set up and execute end-to-end tests using Cypress or Playwright for Angular applications.

## Command Usage

```bash
/angular-testing:run-e2e [cypress|playwright] [setup|run|debug]
```

## Natural Language Examples

- "Set up Cypress for E2E testing"
- "Create E2E tests for login flow"
- "Run Playwright tests in headless mode"
- "Debug failing E2E tests"

## What This Command Does

1. **Framework Setup** - Install and configure Cypress/Playwright
2. **Test Generation** - Create E2E test files for user flows
3. **Custom Commands** - Reusable helpers and utilities
4. **CI/CD Integration** - GitHub Actions/GitLab CI configuration
5. **Debugging** - Tools and strategies for flaky tests

---

## Cypress Setup

### Installation

```bash
npm install --save-dev cypress @cypress/schematic
ng add @cypress/schematic
```

### Configuration

```typescript
// cypress.config.ts
import { defineConfig } from 'cypress';

export default defineConfig({
  e2e: {
    baseUrl: 'http://localhost:4200',
    supportFile: 'cypress/support/e2e.ts',
    specPattern: 'cypress/e2e/**/*.cy.ts',
    viewportWidth: 1280,
    viewportHeight: 720,
    video: true,
    screenshotOnRunFailure: true,
    retries: {
      runMode: 2,
      openMode: 0
    },
    env: {
      apiUrl: 'http://localhost:3000/api'
    }
  }
});
```

### Custom Commands

```typescript
// cypress/support/commands.ts
declare global {
  namespace Cypress {
    interface Chainable {
      login(email: string, password: string): Chainable<void>;
      logout(): Chainable<void>;
      getBySel(selector: string): Chainable<JQuery<HTMLElement>>;
      checkAccessibility(): Chainable<void>;
    }
  }
}

// Login command
Cypress.Commands.add('login', (email: string, password: string) => {
  cy.session([email, password], () => {
    cy.visit('/login');
    cy.getBySel('email-input').type(email);
    cy.getBySel('password-input').type(password);
    cy.getBySel('login-button').click();
    cy.url().should('include', '/dashboard');
    cy.window().its('localStorage.token').should('exist');
  });
});

// Logout command
Cypress.Commands.add('logout', () => {
  cy.clearLocalStorage();
  cy.clearCookies();
});

// Get by data-cy attribute
Cypress.Commands.add('getBySel', (selector: string) => {
  return cy.get(`[data-cy="${selector}"]`);
});

// Accessibility check
Cypress.Commands.add('checkAccessibility', () => {
  cy.injectAxe();
  cy.checkA11y();
});
```

### Example E2E Test

```typescript
// cypress/e2e/login.cy.ts
describe('Login Flow', () => {
  beforeEach(() => {
    cy.visit('/login');
  });

  it('should display login form', () => {
    cy.getBySel('email-input').should('be.visible');
    cy.getBySel('password-input').should('be.visible');
    cy.getBySel('login-button').should('be.visible');
  });

  it('should show validation errors for empty fields', () => {
    cy.getBySel('login-button').click();
    cy.getBySel('email-error').should('contain', 'Email is required');
    cy.getBySel('password-error').should('contain', 'Password is required');
  });

  it('should show error for invalid email format', () => {
    cy.getBySel('email-input').type('invalid-email');
    cy.getBySel('password-input').type('password123');
    cy.getBySel('login-button').click();
    cy.getBySel('email-error').should('contain', 'Invalid email format');
  });

  it('should successfully login with valid credentials', () => {
    cy.intercept('POST', '/api/auth/login', {
      statusCode: 200,
      body: {
        token: 'fake-jwt-token',
        user: { id: 1, email: 'test@example.com' }
      }
    }).as('loginRequest');

    cy.getBySel('email-input').type('test@example.com');
    cy.getBySel('password-input').type('password123');
    cy.getBySel('login-button').click();

    cy.wait('@loginRequest');
    cy.url().should('include', '/dashboard');
    cy.window().its('localStorage.token').should('exist');
  });

  it('should show error message for invalid credentials', () => {
    cy.intercept('POST', '/api/auth/login', {
      statusCode: 401,
      body: { message: 'Invalid credentials' }
    }).as('loginRequest');

    cy.getBySel('email-input').type('test@example.com');
    cy.getBySel('password-input').type('wrong-password');
    cy.getBySel('login-button').click();

    cy.wait('@loginRequest');
    cy.getBySel('error-message').should('contain', 'Invalid credentials');
    cy.url().should('include', '/login');
  });

  it('should toggle password visibility', () => {
    cy.getBySel('password-input').should('have.attr', 'type', 'password');
    cy.getBySel('toggle-password').click();
    cy.getBySel('password-input').should('have.attr', 'type', 'text');
    cy.getBySel('toggle-password').click();
    cy.getBySel('password-input').should('have.attr', 'type', 'password');
  });

  it('should remember me functionality', () => {
    cy.getBySel('remember-me-checkbox').check();
    cy.login('test@example.com', 'password123');
    
    cy.reload();
    cy.url().should('include', '/dashboard');
  });
});
```

### Running Cypress Tests

```bash
# Open Cypress Test Runner
npx cypress open

# Run all tests headless
npx cypress run

# Run specific test
npx cypress run --spec "cypress/e2e/login.cy.ts"

# Run with specific browser
npx cypress run --browser chrome
```

---

## Playwright Setup

### Installation

```bash
npm install --save-dev @playwright/test
npx playwright install
```

### Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:4200',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure'
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] }
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] }
    }
  ],
  webServer: {
    command: 'npm start',
    url: 'http://localhost:4200',
    reuseExistingServer: !process.env.CI
  }
});
```

### Example Playwright Test

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Login Flow', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });

  test('should display login form', async ({ page }) => {
    await expect(page.getByTestId('email-input')).toBeVisible();
    await expect(page.getByTestId('password-input')).toBeVisible();
    await expect(page.getByTestId('login-button')).toBeVisible();
  });

  test('should login successfully', async ({ page }) => {
    await page.route('**/api/auth/login', async route => {
      await route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify({
          token: 'fake-jwt-token',
          user: { id: 1, email: 'test@example.com' }
        })
      });
    });

    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-button"]');

    await expect(page).toHaveURL(/.*dashboard/);
    
    const token = await page.evaluate(() => localStorage.getItem('token'));
    expect(token).toBeTruthy();
  });
});
```

### Running Playwright Tests

```bash
# Run all tests
npx playwright test

# Run with UI
npx playwright test --ui

# Debug mode
npx playwright test --debug

# Generate report
npx playwright show-report
```

---

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on: [push, pull_request]

jobs:
  cypress:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Start Angular dev server
        run: npm start &
        
      - name: Wait for server
        run: npx wait-on http://localhost:4200
      
      - name: Run Cypress tests
        run: npm run cypress:run
      
      - name: Upload screenshots
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: cypress-screenshots
          path: cypress/screenshots
```

## Best Practices

1. **Use data-cy/data-testid attributes** - For stable selectors
2. **Avoid waits** - Use Cypress auto-retry
3. **Test user flows** - Not individual features
4. **Mock external APIs** - For consistency
5. **Keep tests independent** - No shared state
6. **Use fixtures** - For test data
7. **Clean up after tests** - Reset state

## Usage

```bash
/angular-testing:run-e2e

# Examples
"Set up Cypress for testing login flow"
"Create E2E tests for product CRUD operations"
"Run E2E tests in CI/CD pipeline"
```

---

*Comprehensive E2E testing made easy! 🎭*
