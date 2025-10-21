# Audit Security Command

Comprehensive security audit for Angular applications following OWASP guidelines.

## Usage

```bash
/angular-security:audit-security
```

## Audit Categories

### 1. Dependency Vulnerabilities

```bash
# Check for vulnerable packages
npm audit

# Fix automatically
npm audit fix

# View detailed report
npm audit --json

# Check specific severity
npm audit --audit-level=high
```

### 2. XSS Vulnerabilities

```typescript
// Scan for dangerous patterns:
// - innerHTML with user data
// - bypassSecurityTrust* methods
// - [style], [href], [src] with user input
// - Dynamic script/style injection

// ❌ Found issues:
[innerHTML]="userContent"          // XSS risk
[src]="untrustedUrl"               // URL injection
[style]="userStyle"                // Style injection
bypassSecurityTrustHtml(userHtml)  // Bypassing protection
```

### 3. Authentication & Authorization

```typescript
// ✅ Secure auth implementation
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();
  
  if (token) {
    req = req.clone({
      setHeaders: {
        'Authorization': `Bearer ${token}`,
        'X-Requested-With': 'XMLHttpRequest' // CSRF protection
      }
    });
  }
  
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        inject(Router).navigate(['/login']);
      }
      return throwError(() => error);
    })
  );
};

// ✅ Secure route guard
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  if (authService.isAuthenticated()) {
    // Check role-based access
    const requiredRole = route.data['role'];
    if (requiredRole && !authService.hasRole(requiredRole)) {
      return router.parseUrl('/unauthorized');
    }
    return true;
  }
  
  return router.parseUrl('/login');
};
```

### 4. CSRF Protection

```typescript
// ✅ CSRF token implementation
export class CsrfInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Get CSRF token from cookie
    const csrfToken = this.getCookie('XSRF-TOKEN');
    
    // Add to header for state-changing requests
    if (req.method !== 'GET' && req.method !== 'HEAD') {
      req = req.clone({
        setHeaders: {
          'X-XSRF-TOKEN': csrfToken
        }
      });
    }
    
    return next.handle(req);
  }
  
  private getCookie(name: string): string {
    const match = document.cookie.match(new RegExp('(^| )' + name + '=([^;]+)'));
    return match ? match[2] : '';
  }
}
```

### 5. Token Storage

```typescript
// ❌ INSECURE: localStorage
localStorage.setItem('token', jwt);  // Vulnerable to XSS

// ✅ SECURE: HttpOnly cookie (server-side)
// Set-Cookie: token=xxx; HttpOnly; Secure; SameSite=Strict

// ✅ ALTERNATIVE: Memory + refresh token
@Injectable({ providedIn: 'root' })
export class TokenService {
  private accessToken: string | null = null;
  
  setToken(token: string) {
    this.accessToken = token;
    // Refresh token in HttpOnly cookie
  }
  
  getToken(): string | null {
    return this.accessToken;
  }
  
  clearToken() {
    this.accessToken = null;
  }
}
```

### 6. Input Validation

```typescript
// ✅ Comprehensive validation
export class SecureFormComponent {
  form = this.fb.group({
    username: ['', [
      Validators.required,
      Validators.minLength(3),
      Validators.maxLength(20),
      Validators.pattern(/^[a-zA-Z0-9_]+$/)
    ]],
    email: ['', [
      Validators.required,
      Validators.email
    ]],
    age: ['', [
      Validators.required,
      Validators.min(18),
      Validators.max(120)
    ]],
    website: ['', [
      this.urlValidator()
    ]]
  });
  
  private urlValidator(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      
      try {
        const url = new URL(control.value);
        // Only allow https
        if (url.protocol !== 'https:') {
          return { invalidProtocol: true };
        }
        return null;
      } catch {
        return { invalidUrl: true };
      }
    };
  }
}
```

### 7. Secure HTTP Communication

```typescript
// ✅ HTTPS enforcement
export class AppComponent implements OnInit {
  ngOnInit() {
    // Redirect HTTP to HTTPS
    if (location.protocol !== 'https:' && 
        !location.hostname.includes('localhost')) {
      location.replace(`https:${location.href.substring(location.protocol.length)}`);
    }
  }
}

// ✅ Security headers (server-side config)
// Strict-Transport-Security: max-age=31536000; includeSubDomains
// X-Content-Type-Options: nosniff
// X-Frame-Options: DENY
// X-XSS-Protection: 1; mode=block
// Referrer-Policy: strict-origin-when-cross-origin
```

### 8. Content Security Policy

```html
<!-- Strict CSP -->
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self';
  script-src 'self';
  style-src 'self';
  img-src 'self' data: https:;
  font-src 'self';
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
">
```

### 9. Rate Limiting

```typescript
// ✅ Client-side rate limiting
@Injectable({ providedIn: 'root' })
export class RateLimitService {
  private requests = new Map<string, number[]>();
  private readonly WINDOW_MS = 60000; // 1 minute
  private readonly MAX_REQUESTS = 10;
  
  canMakeRequest(key: string): boolean {
    const now = Date.now();
    const timestamps = this.requests.get(key) || [];
    
    // Remove old timestamps
    const validTimestamps = timestamps.filter(ts => 
      now - ts < this.WINDOW_MS
    );
    
    if (validTimestamps.length >= this.MAX_REQUESTS) {
      return false;
    }
    
    validTimestamps.push(now);
    this.requests.set(key, validTimestamps);
    return true;
  }
}
```

### 10. Logging & Monitoring

```typescript
// ✅ Security event logging
@Injectable({ providedIn: 'root' })
export class SecurityLogger {
  logSecurityEvent(event: SecurityEvent) {
    const log = {
      timestamp: new Date().toISOString(),
      event: event.type,
      severity: event.severity,
      user: this.authService.getCurrentUser()?.id,
      details: event.details,
      userAgent: navigator.userAgent,
      ip: this.getClientIp()
    };
    
    // Send to security monitoring service
    this.http.post('/api/security/log', log).subscribe();
  }
  
  logFailedLogin(username: string) {
    this.logSecurityEvent({
      type: 'FAILED_LOGIN',
      severity: 'HIGH',
      details: { username }
    });
  }
  
  logXSSAttempt(payload: string) {
    this.logSecurityEvent({
      type: 'XSS_ATTEMPT',
      severity: 'CRITICAL',
      details: { payload }
    });
  }
}
```

## Audit Report Example

```
🔒 Security Audit Report
Generated: 2025-10-21 15:30:00

┌─────────────────────────────────────────┐
│ OWASP Top 10 Compliance                 │
├─────────────────────────────────────────┤
│ A01: Broken Access Control     ✅ PASS  │
│ A02: Cryptographic Failures    ✅ PASS  │
│ A03: Injection                 ⚠️  WARN │
│ A04: Insecure Design           ✅ PASS  │
│ A05: Security Misconfiguration ❌ FAIL  │
│ A06: Vulnerable Components     ⚠️  WARN │
│ A07: Auth Failures             ✅ PASS  │
│ A08: Software/Data Integrity   ✅ PASS  │
│ A09: Logging Failures          ⚠️  WARN │
│ A10: Server-Side Forgery       ✅ PASS  │
└─────────────────────────────────────────┘

📊 Severity Breakdown:
┌──────────┬───────┬──────────────────┐
│ Severity │ Count │ Issues           │
├──────────┼───────┼──────────────────┤
│ 🔴 CRITICAL │ 0  │                  │
│ 🔴 HIGH     │ 2  │ XSS, CSP missing │
│ 🟡 MEDIUM   │ 5  │ Various          │
│ 🟢 LOW      │ 3  │ Improvements     │
└──────────┴───────┴──────────────────┘

🔴 HIGH Priority Issues:

1. XSS Vulnerability in UserProfileComponent
   File: user-profile.component.ts:45
   Code: [innerHTML]="userBio"
   Fix: Use DomSanitizer or remove innerHTML
   
2. Missing Content-Security-Policy
   File: index.html
   Impact: No XSS protection layer
   Fix: Add CSP meta tag

🟡 MEDIUM Priority Issues:

3. Hardcoded API credentials
   File: environment.ts:8
   Fix: Use environment variables
   
4. No rate limiting on login
   File: auth.service.ts:23
   Fix: Implement rate limiting

5. JWT in localStorage
   File: token.service.ts:12
   Fix: Use HttpOnly cookies

🟢 LOW Priority Issues:

6. Missing security headers
   Fix: Configure server headers
   
7. No input sanitization
   Fix: Add validation to forms

8. Console.log in production
   Fix: Remove debug logs

📋 npm audit Results:
┌──────────┬───────┐
│ Critical │ 0     │
│ High     │ 1     │
│ Moderate │ 3     │
│ Low      │ 8     │
└──────────┴───────┘

Vulnerable packages:
- lodash@4.17.15 (HIGH) → Update to 4.17.21
- axios@0.21.1 (MODERATE) → Update to 1.6.0

✅ Secure Practices Found:
- HTTPS enforced
- Route guards implemented
- Form validation present
- CSRF tokens in use
- Password hashing (bcrypt)

🎯 Security Score: 7.2/10

📈 After Fixes: 9.5/10

⏱️ Estimated Fix Time: 4-6 hours

🔧 Action Plan:
1. Fix XSS vulnerability (30 min)
2. Add CSP headers (15 min)
3. Move JWT to HttpOnly cookie (1 hour)
4. Update vulnerable dependencies (30 min)
5. Add rate limiting (1 hour)
6. Implement security logging (1 hour)
```

## Continuous Security

```yaml
# .github/workflows/security.yml
name: Security Audit

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: npm audit
        run: npm audit --audit-level=moderate
      
      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
      
      - name: Snyk Security Scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

---

*Audit regularly, fix proactively! 🛡️*
