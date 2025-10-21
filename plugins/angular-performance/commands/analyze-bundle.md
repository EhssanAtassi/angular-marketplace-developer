# Analyze Bundle Command

Analyze, visualize, and optimize Angular bundle sizes.

## Usage

```bash
/angular-performance:analyze-bundle
```

## What It Does

1. Generate bundle statistics
2. Visualize bundle composition
3. Identify optimization opportunities
4. Provide actionable recommendations

## Step 1: Generate Bundle Stats

```bash
# Build with stats
ng build --configuration production --stats-json

# Output: dist/<project>/stats.json
```

## Step 2: Analyze with Tools

### Webpack Bundle Analyzer

```bash
npm install --save-dev webpack-bundle-analyzer

# Analyze
npx webpack-bundle-analyzer dist/<project>/stats.json
```

### Source Map Explorer

```bash
npm install --save-dev source-map-explorer

# Build with source maps
ng build --configuration production --source-map

# Analyze
npx source-map-explorer dist/**/*.js
```

## Optimization Strategies

### 1. Lazy Loading

```typescript
// angular.json - before
const routes: Routes = [
  { path: 'dashboard', component: DashboardComponent },
  { path: 'users', component: UsersComponent },
  { path: 'reports', component: ReportsComponent }
];

// After - lazy load feature modules
const routes: Routes = [
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
    path: 'reports',
    loadChildren: () => import('./reports/reports.routes')
      .then(m => m.REPORTS_ROUTES)
  }
];
```

**Impact**: Reduces initial bundle by 40-60%

### 2. Tree Shaking

```typescript
// Before: Imports entire library
import * as _ from 'lodash';

// After: Import only what you need
import { debounce, throttle } from 'lodash-es';

// Or use native alternatives
const debounce = (fn, delay) => {
  let timeoutId;
  return (...args) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
};
```

### 3. Remove Unused Dependencies

```bash
# Find unused dependencies
npx depcheck

# Analyze package size before adding
npx bundlephobia <package-name>
```

### 4. Angular Material - Import Selectively

```typescript
// Before: Import entire Material
import { MatButtonModule } from '@angular/material/button';
import { MatCardModule } from '@angular/material/card';
import { MatTableModule } from '@angular/material/table';
// ... 20+ imports

// After: Only import what you use per component
@Component({
  standalone: true,
  imports: [MatButtonModule], // Only button for this component
  template: '<button mat-raised-button>Click</button>'
})
```

### 5. Replace Heavy Libraries

```typescript
// Before: Moment.js (67KB gzipped)
import * as moment from 'moment';
const date = moment().format('YYYY-MM-DD');

// After: date-fns (6KB gzipped)
import { format } from 'date-fns';
const date = format(new Date(), 'yyyy-MM-dd');

// Or native Intl API (0KB - built-in)
const date = new Intl.DateTimeFormat('en-US').format(new Date());
```

### 6. Code Splitting with Dynamic Imports

```typescript
// Before: Import at top level
import { ChartComponent } from './chart/chart.component';

export class DashboardComponent {
  showChart = false;
  chart = ChartComponent; // Always in bundle
}

// After: Dynamic import
export class DashboardComponent {
  showChart = false;
  chartComponent: any;
  
  async loadChart() {
    const { ChartComponent } = await import('./chart/chart.component');
    this.chartComponent = ChartComponent;
    this.showChart = true;
  }
}
```

### 7. Optimize Images

```bash
# Install image optimization tools
npm install --save-dev imagemin imagemin-webp imagemin-pngquant

# Use WebP format with fallbacks
<picture>
  <source srcset="image.webp" type="image/webp">
  <img src="image.jpg" alt="Description">
</picture>
```

### 8. Enable Build Optimizations

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
        }
      ]
    }
  }
}
```

### 9. Differential Loading

Angular automatically generates ES5 and ES2015+ bundles:

```html
<!-- Modern browsers get smaller ES2015+ bundle -->
<script src="main-es2015.js" type="module"></script>

<!-- Legacy browsers get ES5 bundle -->
<script src="main-es5.js" nomodule></script>
```

### 10. Service Worker & Caching

```bash
# Add service worker
ng add @angular/pwa

# Configures ngsw-config.json for caching
```

## Bundle Budget Enforcement

```json
// angular.json - Set budgets
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
  }
]
```

## Analysis Report Example

```
📦 Bundle Analysis Report

📊 Current Bundle Size:
┌─────────────────┬──────────┬──────────┐
│ Chunk           │ Size     │ Gzipped  │
├─────────────────┼──────────┼──────────┤
│ main.js         │ 1.2 MB   │ 380 KB   │
│ polyfills.js    │ 145 KB   │ 45 KB    │
│ runtime.js      │ 12 KB    │ 4 KB     │
│ styles.css      │ 85 KB    │ 15 KB    │
├─────────────────┼──────────┼──────────┤
│ Total           │ 1.44 MB  │ 444 KB   │
└─────────────────┴──────────┴──────────┘

⚠️ Issues Found:

1. 🔴 Lodash (287 KB) - Replace with lodash-es or native
2. 🟡 Moment.js (67 KB) - Replace with date-fns (6 KB)
3. 🟡 Feature modules not lazy loaded (420 KB in main)
4. 🟡 Angular Material fully imported (180 KB unused)

✅ Recommended Actions:

1. Lazy load feature modules → Save ~400 KB
   - Dashboard, Users, Reports modules

2. Replace heavy libraries:
   - lodash → lodash-es → Save 250 KB
   - moment.js → date-fns → Save 60 KB

3. Tree-shake Material imports → Save 150 KB
   - Import only used components

4. Enable build optimizations → Save 100 KB
   - Already in angular.json

💡 Potential Savings: ~860 KB (59% reduction)

📈 After Optimization:
┌─────────────────┬──────────┬──────────┐
│ main.js         │ 580 KB   │ 180 KB   │
│ dashboard.js    │ 120 KB   │ 35 KB    │ (lazy)
│ users.js        │ 95 KB    │ 28 KB    │ (lazy)
│ reports.js      │ 205 KB   │ 62 KB    │ (lazy)
├─────────────────┼──────────┼──────────┤
│ Total Initial   │ 580 KB   │ 180 KB   │ ⬇️ 60%
│ Total All       │ 1.0 MB   │ 305 KB   │ ⬇️ 31%
└─────────────────┴──────────┴──────────┘
```

## Continuous Monitoring

### GitHub Action for Bundle Size

```yaml
name: Bundle Size Check

on: [pull_request]

jobs:
  check-size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run build -- --stats-json
      
      - name: Analyze bundle
        uses: github/webpack-bundle-analyzer@v1
        with:
          bundle-stats: 'dist/stats.json'
          
      - name: Check size limits
        run: |
          SIZE=$(stat -f%z "dist/main.*.js")
          if [ $SIZE -gt 500000 ]; then
            echo "Bundle too large: $SIZE bytes"
            exit 1
          fi
```

## Quick Wins Checklist

- [ ] Enable production mode optimizations
- [ ] Lazy load all feature modules
- [ ] Replace moment.js with date-fns
- [ ] Use lodash-es instead of lodash
- [ ] Remove unused dependencies
- [ ] Optimize images (WebP, lazy loading)
- [ ] Import Material components selectively
- [ ] Enable differential loading
- [ ] Set up bundle budgets
- [ ] Add service worker for caching

---

*Smaller bundles = Faster apps! 📦⚡*
