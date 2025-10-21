---
name: scaffold-project
description: Generate complete Angular 17+ project structure with Router-First architecture, lazy loading, and enterprise patterns
---

Generate a production-ready Angular 17+ project structure following Router-First methodology.

## What You'll Create

A complete enterprise Angular project with:
- ✅ Router-First architecture
- ✅ Lazy-loaded feature modules
- ✅ Core module (singletons)
- ✅ Shared module (reusables)
- ✅ TypeScript strict mode
- ✅ Standalone components
- ✅ Proper folder organization

## Project Information Needed

Ask the user for:
1. **Project name** - kebab-case (e.g., `my-enterprise-app`)
2. **Key features** - What features does the app need? (e.g., dashboard, users, reports)
3. **Team size** - Small (2-5), Medium (5-20), Large (20+)
4. **Authentication needed?** - Yes/No
5. **Additional requirements** - Any special needs (GIS, real-time, offline)?

## Generation Steps

1. **Create root structure** with app.routes.ts, app.config.ts
2. **Set up core module** with services, guards, interceptors
3. **Create shared module** with common components
4. **Generate feature modules** with lazy loading
5. **Configure TypeScript** with strict mode
6. **Set up routing** with guards and data
7. **Add layout components** (header, sidebar, footer)
8. **Generate example components** for each feature

## Example Output Structure

```
src/app/
├── core/
│   ├── services/
│   │   ├── auth.service.ts
│   │   └── api.service.ts
│   ├── guards/
│   │   └── auth.guard.ts
│   ├── interceptors/
│   │   └── auth.interceptor.ts
│   └── models/
│       └── user.interface.ts
│
├── shared/
│   ├── components/
│   │   ├── loading-spinner/
│   │   └── error-message/
│   └── pipes/
│       └── time-ago.pipe.ts
│
├── features/
│   ├── dashboard/
│   │   ├── components/
│   │   ├── dashboard.routes.ts
│   │   └── dashboard.component.ts
│   └── users/
│       ├── components/
│       ├── users.routes.ts
│       └── users.component.ts
│
├── layout/
│   ├── header/
│   ├── sidebar/
│   └── main-layout.component.ts
│
├── app.routes.ts
├── app.config.ts
└── app.component.ts
```

## Configuration Files to Generate

### tsconfig.json (strict mode)
```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

### angular.json (performance budgets)
```json
{
  "budgets": [
    {
      "type": "initial",
      "maximumWarning": "500kb",
      "maximumError": "1mb"
    }
  ]
}
```

## Best Practices to Include

- All components use `standalone: true`
- All routes use lazy loading except root
- Separate files for templates/styles (no inline)
- Guards use functional style (Angular 14+)
- Services use `inject()` function
- Components use signals for state

## Output Format

Provide:
1. **Complete folder tree** showing all files
2. **Key file contents** (app.routes.ts, core services, example components)
3. **Installation commands** for dependencies
4. **Next steps** for development
5. **Architecture decision rationale** - Why this structure?

## Usage

```bash
# In Claude Code
/angular-architecture:scaffold-project

# Or natural language
"Use angular-architect to scaffold a new e-commerce project with 4 features"
```

## Notes

- Adapt complexity based on team size
- Small teams: simpler structure
- Large teams: more abstraction and tooling
- Always document architectural decisions in README
