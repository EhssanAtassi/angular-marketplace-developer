---
name: create-component
description: Generate a complete Angular 17+ standalone component with template, styles, and optional tests following best practices
---

Generate a production-ready Angular component with all necessary files and configurations.

## Component Types

Ask the user which type:

1. **Smart Component (Container)** - Manages data and business logic
2. **Dumb Component (Presentational)** - Displays data only
3. **Form Component** - Handles form input
4. **Layout Component** - App structure (header, sidebar, etc.)

## Information Needed

1. **Component name** - kebab-case (e.g., `user-profile`)
2. **Component type** - Smart or Dumb
3. **Location** - Feature folder or shared
4. **Include tests?** - Yes/No
5. **Data to display** - What data does it work with?
6. **Actions** - What actions can users perform?

## Generated Files

```
component-name/
├── component-name.component.ts       # TypeScript class
├── component-name.component.html     # Template
├── component-name.component.scss     # Styles
└── component-name.component.spec.ts  # Tests (optional)
```

## Smart Component Template

```typescript
import { Component, signal, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ComponentNameService } from './services/component-name.service';

@Component({
  selector: 'app-component-name',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './component-name.component.html',
  styleUrls: ['./component-name.component.scss']
})
export class ComponentNameComponent {
  private service = inject(ComponentNameService);
  
  data = signal<DataType[]>([]);
  loading = signal(false);
  error = signal<string | null>(null);
  
  ngOnInit() {
    this.loadData();
  }
  
  loadData() {
    this.loading.set(true);
    this.service.getData().subscribe({
      next: data => {
        this.data.set(data);
        this.loading.set(false);
      },
      error: err => {
        this.error.set(err.message);
        this.loading.set(false);
      }
    });
  }
  
  handleAction(id: string) {
    // Handle user action
  }
}
```

## Dumb Component Template

```typescript
import { Component, input, output, ChangeDetectionStrategy } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-component-name',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './component-name.component.html',
  styleUrls: ['./component-name.component.scss'],
  changeDetection: ChangeDetectionStrategy.OnPush  // Performance
})
export class ComponentNameComponent {
  // Modern signal inputs
  data = input.required<DataType>();
  disabled = input(false);
  
  // Modern signal outputs
  action = output<string>();
  delete = output<string>();
  
  handleClick() {
    this.action.emit(this.data().id);
  }
}
```

## Template Structure

### For List Components
```html
<div class="component-name-container">
  @if (loading()) {
    <app-loading-spinner />
  } @else if (error()) {
    <app-error-message [message]="error()" />
  } @else {
    @for (item of data(); track item.id) {
      <div class="item">
        {{ item.name }}
      </div>
    } @empty {
      <p class="empty-state">No items found</p>
    }
  }
</div>
```

### For Detail Components
```html
<div class="component-name-detail">
  @if (data(); as item) {
    <h1>{{ item.title }}</h1>
    <p>{{ item.description }}</p>
    <div class="actions">
      <button (click)="handleEdit()">Edit</button>
      <button (click)="handleDelete()">Delete</button>
    </div>
  }
</div>
```

## Styling Template (SCSS)

```scss
:host {
  display: block;
}

.component-name-container {
  padding: 1rem;
  
  .item {
    margin-bottom: 1rem;
    padding: 1rem;
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    
    &:hover {
      background-color: #f5f5f5;
    }
  }
  
  .empty-state {
    text-align: center;
    color: #666;
    padding: 2rem;
  }
}

// Responsive
@media (max-width: 768px) {
  .component-name-container {
    padding: 0.5rem;
  }
}
```

## Test Template

```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { ComponentNameComponent } from './component-name.component';

describe('ComponentNameComponent', () => {
  let component: ComponentNameComponent;
  let fixture: ComponentFixture<ComponentNameComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ComponentNameComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(ComponentNameComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should display data', () => {
    component.data.set([{ id: '1', name: 'Test' }]);
    fixture.detectChanges();
    
    const compiled = fixture.nativeElement;
    expect(compiled.querySelector('.item')).toBeTruthy();
  });
});
```

## Best Practices to Include

1. **Always standalone: true**
2. **Separate template and style files** (never inline)
3. **OnPush for dumb components**
4. **Use signals for local state**
5. **TrackBy in @for loops**
6. **Proper error handling**
7. **Loading states**
8. **Empty states**
9. **Accessibility attributes**
10. **Responsive design**

## Component Checklist

Generated component should have:
- [ ] Standalone: true
- [ ] Proper imports
- [ ] Separate template file
- [ ] Separate styles file
- [ ] Signal-based state (if smart)
- [ ] Input/Output (if dumb)
- [ ] OnPush (if dumb)
- [ ] TrackBy functions
- [ ] Error handling
- [ ] Loading states
- [ ] Tests (if requested)

## Usage

```bash
# In Claude Code
/angular-development:create-component

# Natural language
"Create a smart component called product-list that displays products"
"Generate a dumb component for user-card with name and email inputs"
```

## Notes

- Smart components go in `features/<feature>/components/`
- Dumb components go in `shared/components/`
- Always ask about data structure before generating
- Include proper TypeScript types
- Add comments for complex logic
