---
name: design-routes
description: Design and plan Angular routes using Router-First methodology before implementing components
---

Design the complete routing structure for an Angular application using Router-First methodology.

## What This Command Does

Following Doguhan Uluca's Router-First approach, this command helps you:
- ✅ Define routes BEFORE building components
- ✅ Plan lazy loading strategy
- ✅ Set up route guards and resolvers
- ✅ Design navigation hierarchy
- ✅ Plan data loading strategies

## Information Needed

Ask the user:
1. **App type** - What kind of application? (e.g., e-commerce, dashboard, CMS)
2. **User roles** - What roles exist? (e.g., admin, user, guest)
3. **Main features** - List all features (e.g., dashboard, products, orders, settings)
4. **Sub-features** - Any nested routes? (e.g., products → list/detail/edit)
5. **Public vs Protected** - Which routes need authentication?

## Router-First Process

### Step 1: Create User Flow Diagram
```
Guest → Login → Dashboard → [Features]
                          ├── Products
                          ├── Orders
                          ├── Users (admin only)
                          └── Settings
```

### Step 2: Design Route Tree
```
/                           (redirect to /dashboard)
/login                      (public, lazy)
/dashboard                  (protected, lazy)
  ├── /overview             (default)
  ├── /analytics            (lazy)
/products                   (protected, lazy)
  ├── /list                 (default)
  ├── /detail/:id           (lazy)
  ├── /create               (lazy, admin only)
/orders                     (protected, lazy)
/users                      (protected, lazy, admin only)
/settings                   (protected, lazy)
```

### Step 3: Define Route Configuration
```typescript
export const routes: Routes = [
  {
    path: '',
    redirectTo: '/dashboard',
    pathMatch: 'full'
  },
  {
    path: 'login',
    loadComponent: () => import('./features/auth/login.component')
      .then(m => m.LoginComponent),
    data: { breadcrumb: 'Login' }
  },
  {
    path: 'dashboard',
    loadChildren: () => import('./features/dashboard/dashboard.routes')
      .then(m => m.DASHBOARD_ROUTES),
    canActivate: [AuthGuard],
    data: { breadcrumb: 'Dashboard' }
  },
  {
    path: 'products',
    loadChildren: () => import('./features/products/products.routes')
      .then(m => m.PRODUCTS_ROUTES),
    canActivate: [AuthGuard],
    data: { breadcrumb: 'Products' }
  },
  {
    path: 'users',
    loadChildren: () => import('./features/users/users.routes')
      .then(m => m.USERS_ROUTES),
    canActivate: [AuthGuard, RoleGuard],
    data: { breadcrumb: 'Users', role: 'admin' }
  }
];
```

### Step 4: Plan Guards and Resolvers
```typescript
// Guards needed
AuthGuard       - Check if user is logged in
RoleGuard       - Check user permissions
UnsavedGuard    - Warn before leaving unsaved forms

// Resolvers needed (optional)
UserResolver    - Load user data before route activates
ProductResolver - Preload product details
```

### Step 5: Loading Strategy
```
Immediate Load (Critical Path):
- Login page
- Dashboard shell
- Error pages

Lazy Load (On Demand):
- Dashboard sub-routes
- Product management
- Admin features
- Reports
- Settings

Preload Strategy:
- Dashboard components (after login)
- High-traffic features
```

## Output Format

Provide:

1. **Visual Route Tree** - ASCII diagram of all routes
2. **Route Configuration Code** - Complete app.routes.ts
3. **Feature Routes** - Example feature routing files
4. **Guards Needed** - List with implementation stubs
5. **Performance Plan** - Bundle size estimates
6. **Navigation Strategy** - How users move through the app
7. **ADR Document** - Architectural Decision Record

## Example Output

### Route Tree Visualization
```
Application Routes (Lazy-Loaded)
│
├─ / (redirect to /dashboard)
│
├─ /login (public)
│
├─ /dashboard (protected, AuthGuard)
│   ├─ /overview (default)
│   ├─ /analytics
│   └─ /reports
│
├─ /products (protected, AuthGuard)
│   ├─ /list (default)
│   ├─ /detail/:id
│   ├─ /create (admin only)
│   └─ /edit/:id (admin only)
│
├─ /orders (protected, AuthGuard)
│   ├─ /list (default)
│   └─ /detail/:id
│
├─ /users (protected, AuthGuard + RoleGuard)
│   ├─ /list (default)
│   ├─ /create
│   └─ /edit/:id
│
├─ /settings (protected, AuthGuard)
│
└─ /** (redirect to /dashboard)
```

### Bundle Size Planning
```
Initial Bundle: 180 KB (gzipped)
- Core services
- Layout components
- Router

Feature Bundles:
- Dashboard: 45 KB
- Products: 60 KB (data grid)
- Orders: 35 KB
- Users: 40 KB
- Settings: 25 KB

Total: ~385 KB (all features loaded)
```

## Best Practices

When designing routes:
- ✅ Keep URLs clean and RESTful
- ✅ Use kebab-case for paths
- ✅ Lazy load everything except critical path
- ✅ Group related features
- ✅ Plan for future features
- ✅ Add breadcrumb data
- ✅ Consider SEO (if applicable)
- ✅ Handle 404s gracefully

## Common Patterns

### Nested Routes
```typescript
{
  path: 'products',
  component: ProductsLayoutComponent,
  children: [
    { path: '', component: ProductListComponent },
    { path: ':id', component: ProductDetailComponent },
    { path: ':id/edit', component: ProductEditComponent }
  ]
}
```

### Route Guards
```typescript
{
  path: 'admin',
  canActivate: [AuthGuard, AdminGuard],
  canDeactivate: [UnsavedChangesGuard],
  children: [...]
}
```

### Route Data
```typescript
{
  path: 'dashboard',
  data: {
    breadcrumb: 'Dashboard',
    title: 'Dashboard - My App',
    animation: 'DashboardPage'
  }
}
```

## Usage

```bash
# In Claude Code
/angular-architecture:design-routes

# Or natural language
"Design routes for an e-commerce app with products, cart, checkout, and admin"
"Use angular-architect to plan routing for a real estate listing platform"
```

## Deliverables

After running this command, you'll have:
1. Complete route structure documented
2. Code for app.routes.ts
3. Feature route files
4. Guard implementations
5. Performance plan
6. Next implementation steps

## Notes

- Routes should tell a story of your app's features
- Design for scalability (easy to add new features)
- Consider analytics tracking in routing
- Plan error handling routes (404, 403, 500)
