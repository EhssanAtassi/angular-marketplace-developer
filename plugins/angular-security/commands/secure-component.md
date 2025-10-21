# Secure Component Command

Analyze and secure Angular components against XSS, injection, and other vulnerabilities.

## Usage

```bash
/angular-security:secure-component <ComponentName>
```

## Security Checks

### 1. XSS Prevention

```typescript
// ❌ CRITICAL: XSS vulnerability
@Component({
  template: `<div [innerHTML]="userInput"></div>`
})
export class UnsafeComponent {
  @Input() userInput: string;
}

// ✅ SECURE: Sanitized HTML
@Component({
  template: `<div [innerHTML]="safeContent"></div>`
})
export class SafeComponent {
  @Input() set userInput(value: string) {
    this.safeContent = this.sanitizer.sanitize(
      SecurityContext.HTML,
      value
    );
  }
  
  safeContent: SafeHtml;
  
  constructor(private sanitizer: DomSanitizer) {}
}

// ✅ BEST: Avoid innerHTML
@Component({
  template: `<div>{{ userInput }}</div>` // Auto-escaped
})
export class BestComponent {
  @Input() userInput: string;
}
```

### 2. URL Sanitization

```typescript
// ❌ DANGEROUS: Arbitrary URL
<a [href]="userUrl">Click</a>

// ✅ SECURE: Sanitize URLs
export class LinkComponent {
  @Input() set url(value: string) {
    this.safeUrl = this.sanitizer.sanitize(
      SecurityContext.URL,
      value
    );
  }
  
  safeUrl: SafeUrl;
  
  constructor(private sanitizer: DomSanitizer) {}
}

// Template
<a [href]="safeUrl">Click</a>
```

### 3. Resource URL Validation

```typescript
// ❌ DANGEROUS: Untrusted resource
<iframe [src]="videoUrl"></iframe>

// ✅ SECURE: Whitelist trusted domains
export class VideoComponent {
  @Input() set videoUrl(url: string) {
    if (this.isTrustedDomain(url)) {
      this.safeUrl = this.sanitizer.bypassSecurityTrustResourceUrl(url);
    } else {
      console.error('Untrusted video URL:', url);
      this.safeUrl = null;
    }
  }
  
  safeUrl: SafeResourceUrl | null;
  
  private trustedDomains = ['youtube.com', 'vimeo.com'];
  
  private isTrustedDomain(url: string): boolean {
    try {
      const domain = new URL(url).hostname;
      return this.trustedDomains.some(trusted => 
        domain.includes(trusted)
      );
    } catch {
      return false;
    }
  }
  
  constructor(private sanitizer: DomSanitizer) {}
}
```

### 4. Style Injection

```typescript
// ❌ DANGEROUS: User-controlled styles
<div [style]="userStyle">Content</div>

// ✅ SECURE: Sanitize styles
export class StyledComponent {
  @Input() set userStyle(value: string) {
    this.safeStyle = this.sanitizer.sanitize(
      SecurityContext.STYLE,
      value
    );
  }
  
  safeStyle: SafeStyle;
  
  constructor(private sanitizer: DomSanitizer) {}
}

// ✅ BETTER: Use class binding
<div [class]="userClass">Content</div>
```

### 5. Dynamic Script Loading

```typescript
// ❌ NEVER DO THIS
eval(userCode);
new Function(userCode)();

// ✅ SECURE: No dynamic code execution
// Use configuration objects instead
export class ConfigurableComponent {
  @Input() config: {
    enabled: boolean;
    options: string[];
  };
  
  executeAction() {
    // Safe configuration-driven logic
    if (this.config.enabled) {
      this.config.options.forEach(opt => this.handleOption(opt));
    }
  }
}
```

### 6. Form Input Validation

```typescript
// ✅ SECURE: Comprehensive validation
export class UserFormComponent {
  userForm = this.fb.group({
    username: ['', [
      Validators.required,
      Validators.minLength(3),
      Validators.maxLength(20),
      Validators.pattern(/^[a-zA-Z0-9_]+$/) // Alphanumeric only
    ]],
    email: ['', [
      Validators.required,
      Validators.email
    ]],
    bio: ['', [
      Validators.maxLength(500)
    ]]
  });
  
  constructor(private fb: FormBuilder) {}
  
  onSubmit() {
    if (this.userForm.valid) {
      // Server-side validation still required!
      const data = this.userForm.value;
      this.api.createUser(data).subscribe();
    }
  }
}
```

### 7. File Upload Security

```typescript
// ✅ SECURE: File validation
export class FileUploadComponent {
  private readonly MAX_SIZE = 5 * 1024 * 1024; // 5MB
  private readonly ALLOWED_TYPES = [
    'image/jpeg',
    'image/png',
    'image/gif',
    'application/pdf'
  ];
  
  onFileSelected(event: Event) {
    const input = event.target as HTMLInputElement;
    const file = input.files?.[0];
    
    if (!file) return;
    
    // Validate file type
    if (!this.ALLOWED_TYPES.includes(file.type)) {
      alert('Invalid file type');
      return;
    }
    
    // Validate file size
    if (file.size > this.MAX_SIZE) {
      alert('File too large (max 5MB)');
      return;
    }
    
    // Validate file extension
    const ext = file.name.split('.').pop()?.toLowerCase();
    const validExtensions = ['jpg', 'jpeg', 'png', 'gif', 'pdf'];
    if (!ext || !validExtensions.includes(ext)) {
      alert('Invalid file extension');
      return;
    }
    
    this.uploadFile(file);
  }
}
```

### 8. Sensitive Data in Templates

```typescript
// ❌ DANGEROUS: Exposing sensitive data
@Component({
  template: `
    <div>API Key: {{ apiKey }}</div>
    <div>Password: {{ password }}</div>
  `
})

// ✅ SECURE: Never expose sensitive data
@Component({
  template: `
    <div>Status: {{ isAuthenticated ? 'Connected' : 'Disconnected' }}</div>
  `
})
export class SecureComponent {
  isAuthenticated: boolean;
  
  // Sensitive data only in memory, never in template
  private apiKey: string;
  private password: string;
}
```

### 9. Clickjacking Prevention

```typescript
// Set X-Frame-Options header (server-side)
// But also check in Angular:

export class FrameGuard implements OnInit {
  ngOnInit() {
    // Prevent loading in iframe
    if (window.self !== window.top) {
      console.error('Clickjacking attempt detected');
      window.top.location = window.self.location;
    }
  }
}
```

### 10. Content Security Policy

```html
<!-- index.html -->
<meta http-equiv="Content-Security-Policy" 
      content="
        default-src 'self';
        script-src 'self' 'unsafe-inline';
        style-src 'self' 'unsafe-inline';
        img-src 'self' data: https:;
        font-src 'self' data:;
        connect-src 'self' https://api.example.com;
        frame-ancestors 'none';
      ">
```

## Security Audit Checklist

```typescript
// Component security audit
export class AuditedComponent {
  // ✅ 1. No innerHTML with user data
  // ✅ 2. All URLs sanitized
  // ✅ 3. Form inputs validated
  // ✅ 4. No eval or Function constructor
  // ✅ 5. Sensitive data not in template
  // ✅ 6. File uploads validated
  // ✅ 7. External resources whitelisted
  // ✅ 8. CSP headers configured
  // ✅ 9. HTTPS enforced
  // ✅ 10. Dependencies audited
}
```

## Output Example

```
🔒 Security Audit: UserProfileComponent

⚠️ Issues Found:

1. 🔴 CRITICAL: XSS Vulnerability (Line 23)
   Location: template: `<div [innerHTML]="bio"></div>`
   Risk: Arbitrary JavaScript execution
   Fix: Use sanitizer or remove innerHTML
   
2. 🟡 HIGH: Unvalidated File Upload (Line 45)
   Location: onFileUpload(event)
   Risk: Malicious file upload
   Fix: Add file type and size validation
   
3. 🟡 MEDIUM: Hardcoded API Key (Line 12)
   Location: apiKey = 'sk_live_123...'
   Risk: Credential exposure
   Fix: Use environment variables

✅ Secure Patterns Detected:
- Form validation with Validators
- HTTPS enforced
- HttpOnly cookies for auth token

📋 Recommendations:
1. Sanitize HTML content
2. Validate file uploads
3. Move API key to environment config
4. Add CSP headers
5. Implement rate limiting for API calls

🎯 Security Score: 6/10 (Medium Risk)
After fixes: 9/10 (Low Risk)
```

---

*Secure every component! 🛡️*
