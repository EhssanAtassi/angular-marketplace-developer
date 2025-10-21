# Angular Master Marketplace

Enterprise-grade Angular 17+ development marketplace with specialized agents for architecture, development, testing, performance, security, and GIS integration.

**Based on:** "Angular for Enterprise Applications, 3rd Edition" by Doguhan Uluca & Official Angular Documentation

---

## 🎯 What's This?

A **Claude Code marketplace** that transforms the comprehensive Angular-Master agent into **6 focused, specialized plugins** following the Seth Hobson pattern.

Each plugin is:
- ✅ **Single-purpose** - Does one thing extremely well
- ✅ **Token-efficient** - Loads only what you need
- ✅ **Production-ready** - Based on real-world patterns
- ✅ **Router-First** - Follows Doguhan Uluca's methodology

---

## 📦 Available Plugins

### 1. **angular-architecture**
Router-First architecture design, enterprise patterns, project structure planning.

**Agent:** `angular-architect` (opus)
**Commands:** `scaffold-project`, `design-routes`
**Skills:** `router-first-methodology`, `enterprise-patterns`

### 2. **angular-development**
Modern Angular 17+ development with standalone components, signals, and RxJS.

**Agent:** `angular-developer` (sonnet)
**Commands:** `create-component`, `create-service`
**Skills:** `signals-patterns`, `rxjs-operators`

### 3. **angular-testing**
Comprehensive testing with Jasmine/Jest, TestBed, E2E with Cypress/Playwright.

**Agent:** `angular-tester` (sonnet)
**Commands:** `generate-tests`, `run-e2e`
**Skills:** `testing-strategies`, `mocking-patterns`

### 4. **angular-performance**
Performance optimization with OnPush, lazy loading, @defer, and Core Web Vitals.

**Agent:** `angular-performance-engineer` (opus)
**Commands:** `analyze-bundle`, `optimize-app`
**Skills:** `change-detection`, `lazy-loading-patterns`

### 5. **angular-security**
Security patterns with XSS prevention, CSRF protection, CSP, and authentication.

**Agent:** `angular-security-specialist` (opus)
**Commands:** `security-audit`, `setup-auth`
**Skills:** `security-checklist`, `auth-patterns`

### 6. **angular-gis**
GIS/geospatial development with PostGIS, Leaflet, OpenLayers, spatial queries.

**Agent:** `angular-gis-specialist` (opus)
**Commands:** `create-map`, `spatial-query`
**Skills:** `postgis-integration`, `leaflet-patterns`

---

## 🚀 Installation

### 1. Add the Marketplace
```bash
/plugin marketplace add YOUR_GITHUB_USERNAME/angular-master-marketplace
```

### 2. Install Plugins
```bash
# Install all
/plugin install angular-architecture
/plugin install angular-development
/plugin install angular-testing
/plugin install angular-performance
/plugin install angular-security
/plugin install angular-gis

# Or just what you need
/plugin install angular-development
/plugin install angular-testing
```

### 3. Use Natural Language or Commands

**Natural Language:**
```
"Use angular-architect to design routes for an e-commerce app"
"Use angular-developer to create a user dashboard component"
"Use angular-gis-specialist to integrate Leaflet with PostGIS"
```

**Slash Commands:**
```bash
/angular-architecture:scaffold-project my-enterprise-app
/angular-development:create-component user-profile
/angular-testing:generate-tests user-profile
/angular-gis:create-map
```

---

## 📚 Core Philosophy

### 1. Router-First Architecture (Uluca Method)
- ✅ Define routes BEFORE components
- ✅ Plan for lazy loading from day one
- ✅ Enforce stateless, data-driven design

### 2. No Inline Templates/Styles (Strict Rule)
- ✅ Always use `templateUrl` and `styleUrls`
- ✅ Separate concerns for maintainability

### 3. Standalone Components (Angular 17+ Default)
- ✅ Modern Angular with standalone-first
- ✅ Tree-shakable and efficient

### 4. TypeScript Strict Mode (No Compromises)
- ✅ `strict: true` in tsconfig.json
- ✅ No `any` types allowed

### 5. Reactive Programming (Observables + Signals)
- ✅ Prefer `async` pipe over manual subscriptions
- ✅ Use signals for reactive state

---

## 🏗️ Project Structure

```
src/app/
├── core/                # Singleton services, guards, interceptors
├── shared/              # Reusable components, directives, pipes
├── features/            # Lazy-loaded feature modules
│   ├── dashboard/
│   ├── users/
│   └── map-viewer/      # GIS example
├── layout/              # App shell components
├── app.routes.ts        # Root routing
└── app.config.ts        # App providers
```

---

## 🎓 Who Is This For?

- **Enterprise teams** (5-100+ developers)
- **Full-stack developers** building Angular apps
- **GIS specialists** integrating maps and spatial data
- **Architects** planning scalable applications
- **Anyone** following modern Angular best practices

---

## 📖 Resources

**Official:**
- [Angular Documentation](https://angular.dev)
- [Angular Style Guide](https://angular.dev/style-guide)
- [RxJS Documentation](https://rxjs.dev)

**Books:**
- "Angular for Enterprise Applications, 3rd Edition" by Doguhan Uluca

**Patterns:**
- Router-First Architecture
- Smart/Dumb Components
- OnPush Change Detection
- Reactive Forms

---

## 🛠️ Development Status

**Version:** 1.0.0  
**Status:** ✅ Initial Release  
**Angular Version:** 17+  
**Last Updated:** October 2025

---

## 📝 License

MIT License - See LICENSE file for details

---

## 🤝 Contributing

This marketplace is built from the Angular-Master agent by Ihsan, combining:
- Doguhan Uluca's Router-First methodology
- Official Angular team best practices
- Real-world enterprise patterns
- GIS integration expertise

---

**Made with ❤️ for the Angular community**
