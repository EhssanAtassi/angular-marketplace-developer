# Angular Bundle Optimization

Complete guide to reducing Angular bundle sizes through code splitting, tree shaking, and lazy loading.

## Table of Contents

1. [Bundle Analysis](#bundle-analysis)
2. [Lazy Loading Strategies](#lazy-loading-strategies)
3. [Tree Shaking](#tree-shaking)
4. [Code Splitting](#code-splitting)
5. [Library Optimization](#library-optimization)
6. [Build Configuration](#build-configuration)
7. [Image Optimization](#image-optimization)
8. [Caching Strategies](#caching-strategies)

---

## Bundle Analysis

### Generate Bundle Stats

```bash
# Build with statistics
ng build --configuration production --stats-json

# Output: dist/<project>/stats.json
```

### Analyze with Tools

```bash
# Webpack Bundle Analyzer
npm install --save-dev webpack-bundle-analyzer
npx webpack-bundle-analyzer dist/<project>/stats.json

# Source Map Explorer
npm install --save-dev source-map-explorer
ng build --configuration production --source-map
npx source-map-explorer dist/**/*.js

# Bundle Buddy
npx bundle-buddy dist/<project>/stats.json
```

### Reading the Analysis

```
main.js (1.2 MB)
├── @angular/core (280 KB)
├── @angular/common (150 KB)
├── lodash (287 KB) ⚠️ Can optimize
├── moment (67 KB) ⚠️ Can replace
├── rxjs (98 KB)
└── application code (318 KB)
```

---

## Lazy Loading Strategies

### Route-Based Lazy Loading

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard/dashboard.component')
      .then(m => m.DashboardComponent)
  },
  {
    path: 'users',
    loadChildren: () => import('./users/users.routes')
      .then(m => m.USERS_ROUTES)
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes')
      .then(m => m.ADMIN_ROUTES),
    canActivate: [AuthGuard] // Only loads if authorized
  }
];
```

**Impact**: Main bundle reduced by 40-60%

### Component Lazy Loading

```typescript
// Before: Imported at module level
import { HeavyChartComponent } from './chart/heavy-chart.component';

@Component({
  template: `
    <app-heavy-chart *ngIf="showChart" [data]="chartData" />
  `
})
export class DashboardComponent {
  showChart = false;
}

// After: Dynamic import
@Component({
  template: `
    <ng-container *ngIf="chartComponent">
      <ng-container *ngComponentOutlet="chartComponent; inputs: chartInputs" />
    </ng-container>
  `
})
export class DashboardComponent {
  chartComponent: any;
  chartInputs = { data: [] };
  
  async loadChart() {
    const { HeavyChartComponent } = await import('./chart/heavy-chart.component');
    this.chartComponent = HeavyChartComponent;
  }
}
```

### Preloading Strategies

```typescript
// app.config.ts
import { PreloadAllModules, NoPreloading } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      // Option 1: No preloading (default)
      withPreloading(NoPreloading),
      
      // Option 2: Preload all
      withPreloading(PreloadAllModules),
      
      // Option 3: Custom preloading
      withPreloading(CustomPreloadingStrategy)
    )
  ]
};

// Custom preloading strategy
@Injectable({ providedIn: 'root' })
export class CustomPreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Preload only routes with data.preload = true
    return route.data?.['preload'] ? load() : of(null);
  }
}

// Route configuration
{
  path: 'important',
  loadChildren: () => import('./important/routes'),
  data: { preload: true } // Will preload
}
```

### Lazy Load on Interaction

```typescript
@Component({
  template: `
    <button (click)="openDialog()">Open Settings</button>
  `
})
export class AppComponent {
  async openDialog() {
    // Load dialog only when button clicked
    const { SettingsDialogComponent } = await import(
      './settings/settings-dialog.component'
    );
    
    const dialogRef = this.dialog.open(SettingsDialogComponent);
  }
}
```

---

## Tree Shaking

### How Tree Shaking Works

Tree shaking removes unused code during the build process.

```typescript
// library.ts
export function usedFunction() { /* ... */ }
export function unusedFunction() { /* ... */ }

// app.ts
import { usedFunction } from './library';

usedFunction();
// unusedFunction is removed from bundle ✂️
```

### Optimize Imports

```typescript
// ❌ BAD: Imports entire library
import * as _ from 'lodash';
import * as moment from 'moment';

_.debounce(fn, 300);
moment().format('YYYY-MM-DD');

// ✅ GOOD: Import only what you need
import { debounce } from 'lodash-es';
import { format } from 'date-fns';

debounce(fn, 300);
format(new Date(), 'yyyy-MM-dd');
```

### Material Components

```typescript
// ❌ BAD: Import entire Material module
import { MatButtonModule } from '@angular/material/button';
import { MatCardModule } from '@angular/material/card';
import { MatTableModule } from '@angular/material/table';
import { MatPaginatorModule } from '@angular/material/paginator';
// ... 20+ more imports in a shared module

// ✅ GOOD: Import per component/feature
@Component({
  standalone: true,
  imports: [
    MatButtonModule, // Only button needed here
    CommonModule
  ]
})
export class SimpleComponent { }
```

### providedIn: 'root' for Services

```typescript
// ❌ BAD: Service in module providers
@NgModule({
  providers: [DataService] // Always in bundle
})

// ✅ GOOD: Tree-shakeable service
@Injectable({
  providedIn: 'root' // Only in bundle if used
})
export class DataService { }
```

### Side-Effect-Free Code

```typescript
// ❌ BAD: Side effects prevent tree shaking
export class Logger {
  constructor() {
    console.log('Logger initialized'); // Side effect!
  }
}

// Even if unused, stays in bundle

// ✅ GOOD: No side effects
export class Logger {
  log(message: string) {
    console.log(message);
  }
}
```

---

## Code Splitting

### Manual Chunks

```typescript
// angular.json
{
  "projects": {
    "app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "optimization": {
                "scripts": true,
                "styles": {
                  "minify": true
                }
              },
              "budgets": [
                {
                  "type": "initial",
                  "maximumWarning": "500kb",
                  "maximumError": "1mb"
                }
              ],
              "namedChunks": false,
              "outputHashing": "all"
            }
          }
        }
      }
    }
  }
}
```

### Vendor Chunking

Angular automatically creates vendor chunks:

```
dist/
├── main.js        (Your code)
├── vendor.js      (node_modules)
├── polyfills.js   (Browser polyfills)
└── runtime.js     (Webpack runtime)
```

### Custom Webpack Config

```typescript
// custom-webpack.config.ts
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          priority: 10
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true
        },
        // Separate heavy libraries
        charts: {
          test: /[\\/]node_modules[\\/](chart\.js|d3)[\\/]/,
          name: 'charts',
          priority: 15
        }
      }
    }
  }
};

// angular.json
{
  "architect": {
    "build": {
      "builder": "@angular-builders/custom-webpack:browser",
      "options": {
        "customWebpackConfig": {
          "path": "./custom-webpack.config.ts"
        }
      }
    }
  }
}
```

---

## Library Optimization

### Replace Heavy Libraries

```typescript
// ❌ Moment.js (67 KB gzipped)
import * as moment from 'moment';
const date = moment().format('YYYY-MM-DD');

// ✅ date-fns (6 KB gzipped)
import { format } from 'date-fns';
const date = format(new Date(), 'yyyy-MM-dd');

// ✅ Native Intl API (0 KB - built-in)
const date = new Intl.DateTimeFormat('en-US', {
  year: 'numeric',
  month: '2-digit',
  day: '2-digit'
}).format(new Date());

// ❌ Lodash (24 KB gzipped)
import _ from 'lodash';

// ✅ Lodash-es (tree-shakeable)
import { debounce, throttle } from 'lodash-es';

// ✅ Native alternatives
const unique = [...new Set(array)];
const grouped = Object.groupBy(array, item => item.category);
```

### Polyfill Only What's Needed

```typescript
// polyfills.ts
// ❌ Import everything
import 'core-js';

// ✅ Import selectively
import 'core-js/es/array/flat';
import 'core-js/es/array/flat-map';
import 'core-js/es/string/replace-all';

// Or use browserslist to auto-determine
```

### Check Before Adding

```bash
# Check package size before installing
npx bundlephobia <package-name>

# Example output:
# lodash: 72.4 KB (24.2 KB gzipped)
# lodash-es: 72.4 KB (tree-shakeable!)
# date-fns: 78 KB (6 KB gzipped per function)
```

---

## Build Configuration

### Production Optimizations

```json
// angular.json
{
  "configurations": {
    "production": {
      "optimization": {
        "scripts": true,
        "styles": {
          "minify": true,
          "inlineCritical": true
        },
        "fonts": true
      },
      "outputHashing": "all",
      "sourceMap": false,
      "namedChunks": false,
      "aot": true,
      "extractLicenses": true,
      "buildOptimizer": true,
      "fileReplacements": [
        {
          "replace": "src/environments/environment.ts",
          "with": "src/environments/environment.prod.ts"
        }
      ]
    }
  }
}
```

### Bundle Budgets

```json
{
  "budgets": [
    {
      "type": "initial",
      "maximumWarning": "500kb",
      "maximumError": "1mb"
    },
    {
      "type": "anyComponentStyle",
      "maximumWarning": "6kb",
      "maximumError": "10kb"
    },
    {
      "type": "bundle",
      "name": "vendor",
      "maximumWarning": "2mb",
      "maximumError": "3mb"
    }
  ]
}
```

### Differential Loading

Angular automatically generates ES5 and ES2015+ bundles:

```html
<!-- Modern browsers (95% of users) -->
<script src="main-es2015.js" type="module"></script>

<!-- Legacy browsers (5% of users) -->
<script src="main-es5.js" nomodule></script>
```

**Impact**: Modern browsers download 30-40% less code

### AOT Compilation

```json
// Always use AOT in production
{
  "aot": true, // Ahead-of-Time compilation
  "buildOptimizer": true // Additional optimizations
}
```

**Benefits**:
- Faster rendering (templates pre-compiled)
- Smaller bundles (compiler not included)
- Early error detection

---

## Image Optimization

### Modern Formats

```html
<!-- Use WebP with fallback -->
<picture>
  <source srcset="image.webp" type="image/webp">
  <source srcset="image.jpg" type="image/jpeg">
  <img src="image.jpg" alt="Description">
</picture>
```

### Responsive Images

```html
<img
  srcset="
    image-320w.jpg 320w,
    image-640w.jpg 640w,
    image-1280w.jpg 1280w
  "
  sizes="(max-width: 320px) 280px,
         (max-width: 640px) 600px,
         1200px"
  src="image-640w.jpg"
  alt="Description"
>
```

### Lazy Loading

```html
<!-- Native lazy loading -->
<img src="image.jpg" loading="lazy" alt="Description">

<!-- With Angular -->
<img 
  [src]="imageUrl" 
  loading="lazy"
  [width]="300"
  [height]="200"
  alt="Description"
>
```

### Image CDN

```typescript
// Use ImageKit, Cloudinary, or similar
const optimizedUrl = `https://ik.imagekit.io/demo/tr:w-400,h-300,f-webp/${imagePath}`;

// Transformations:
// tr:w-400,h-300 = resize to 400x300
// f-webp = convert to WebP
// q-80 = quality 80%
```

---

## Caching Strategies

### Service Worker

```bash
# Add Angular PWA
ng add @angular/pwa
```

```json
// ngsw-config.json
{
  "index": "/index.html",
  "assetGroups": [
    {
      "name": "app",
      "installMode": "prefetch",
      "resources": {
        "files": [
          "/favicon.ico",
          "/index.html",
          "/*.css",
          "/*.js"
        ]
      }
    },
    {
      "name": "assets",
      "installMode": "lazy",
      "updateMode": "prefetch",
      "resources": {
        "files": [
          "/assets/**",
          "/*.(eot|svg|cur|jpg|png|webp|gif|otf|ttf|woff|woff2)"
        ]
      }
    }
  ],
  "dataGroups": [
    {
      "name": "api",
      "urls": ["/api/**"],
      "cacheConfig": {
        "maxSize": 100,
        "maxAge": "1h",
        "timeout": "10s",
        "strategy": "freshness"
      }
    }
  ]
}
```

### HTTP Caching Headers

```typescript
// HTTP interceptor
export const cacheInterceptor: HttpInterceptorFn = (req, next) => {
  // Add cache headers for static assets
  if (req.url.includes('/api/static/')) {
    req = req.clone({
      setHeaders: {
        'Cache-Control': 'public, max-age=31536000, immutable'
      }
    });
  }
  
  return next(req);
};
```

### Browser Caching

```typescript
// Output hashing (angular.json)
{
  "outputHashing": "all" // Generates unique filenames
}

// Result:
// main.abc123.js
// vendor.def456.js
// Enables long-term caching!
```

---

## Optimization Checklist

### Quick Wins
- [ ] Enable production mode
- [ ] Lazy load all feature routes
- [ ] Use lodash-es instead of lodash
- [ ] Replace moment.js with date-fns
- [ ] Enable OnPush change detection
- [ ] Add service worker for caching
- [ ] Optimize images (WebP, lazy loading)

### Medium Impact
- [ ] Analyze bundle with webpack-bundle-analyzer
- [ ] Remove unused dependencies
- [ ] Import Material components selectively
- [ ] Set up bundle budgets
- [ ] Implement preloading strategy
- [ ] Use trackBy for lists
- [ ] Add virtual scrolling for large lists

### Advanced
- [ ] Custom Webpack configuration
- [ ] Image CDN integration
- [ ] HTTP/2 push
- [ ] Critical CSS inlining
- [ ] SSR/Prerendering
- [ ] Compression (Brotli/Gzip)

---

## Measuring Success

### Before Optimization
```
Bundle Sizes:
- main.js: 1.2 MB (380 KB gzipped)
- vendor.js: 850 KB (280 KB gzipped)
- Total: 2.05 MB (660 KB gzipped)

Lighthouse Score: 67
- FCP: 2.1s
- LCP: 3.5s
- TTI: 4.2s
```

### After Optimization
```
Bundle Sizes:
- main.js: 420 KB (135 KB gzipped)
- vendor.js: 580 KB (180 KB gzipped)
- dashboard.js: 120 KB (lazy)
- users.js: 95 KB (lazy)
- Total Initial: 1 MB (315 KB gzipped)

Lighthouse Score: 98
- FCP: 0.8s
- LCP: 1.2s
- TTI: 1.5s

Improvement:
- Bundle size: ⬇️ 51% reduction
- Load time: ⬇️ 64% faster
- Performance score: ⬆️ 46% better
```

---

## Tools Reference

```bash
# Analysis
npx webpack-bundle-analyzer dist/stats.json
npx source-map-explorer dist/**/*.js
npx bundlephobia <package>

# Building
ng build --configuration production --stats-json
ng build --source-map

# Testing
npm run lighthouse
npx @unlighthouse/cli --site http://localhost:4200
```

---

*Optimize bundles for lightning-fast loads! ⚡📦*
