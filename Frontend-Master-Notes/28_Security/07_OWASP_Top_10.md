# OWASP Top 10 Frontend Vulnerabilities - Complete Guide

## Core Concepts

The OWASP (Open Web Application Security Project) Top 10 represents the most critical security risks for web applications. This guide focuses on frontend-specific vulnerabilities and their mitigations with real-world examples.

## 1. Broken Access Control

Access control enforces policies such that users cannot act outside their intended permissions. Failures typically lead to unauthorized information disclosure, modification, or destruction.

### Common Vulnerabilities

```typescript
// ❌ BAD: Client-side only access control
function AdminDashboard() {
  const user = getCurrentUser();

  // VULNERABLE: Attacker can modify localStorage
  if (localStorage.getItem('isAdmin') === 'true') {
    return <AdminPanel />;
  }

  return <AccessDenied />;
}

// ❌ BAD: Exposing sensitive data in frontend state
interface User {
  id: string;
  name: string;
  email: string;
  role: string;
  salary: number; // SHOULD NOT BE IN FRONTEND
  ssn: string; // SHOULD NOT BE IN FRONTEND
}

// ✅ GOOD: Server-side validation + token-based access
class AccessControlManager {
  private token: string | null = null;

  async makeAuthorizedRequest<T>(
    url: string,
    requiredRole?: string
  ): Promise<T> {
    const headers = new Headers({
      'Authorization': `Bearer ${this.token}`,
    });

    // Server validates token and role
    const response = await fetch(url, { headers });

    if (response.status === 403) {
      throw new Error('Insufficient permissions');
    }

    if (!response.ok) {
      throw new Error('Request failed');
    }

    return await response.json();
  }

  // Check permissions on server
  async checkPermission(resource: string, action: string): Promise<boolean> {
    const response = await fetch('/api/auth/check-permission', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.token}`,
      },
      body: JSON.stringify({ resource, action }),
    });

    const result = await response.json();
    return result.allowed;
  }

  setToken(token: string): void {
    this.token = token;
  }
}

// React component with proper access control
function ProtectedRoute({ requiredRole, children }: ProtectedRouteProps) {
  const [hasAccess, setHasAccess] = useState<boolean | null>(null);
  const accessControl = new AccessControlManager();

  useEffect(() => {
    const checkAccess = async () => {
      try {
        const allowed = await accessControl.checkPermission('route', requiredRole);
        setHasAccess(allowed);
      } catch {
        setHasAccess(false);
      }
    };

    checkAccess();
  }, [requiredRole]);

  if (hasAccess === null) return <Loading />;
  if (!hasAccess) return <Navigate to="/access-denied" />;

  return <>{children}</>;
}

interface ProtectedRouteProps {
  requiredRole: string;
  children: React.ReactNode;
}
```

### Real-World Example: Resource Access Control

```typescript
// Resource-based access control
class ResourceAccessControl {
  private permissions: Map<string, Permission[]> = new Map();

  async canAccess(
    userId: string,
    resourceId: string,
    action: Action,
  ): Promise<boolean> {
    const response = await fetch("/api/permissions/check", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ userId, resourceId, action }),
    });

    const result = await response.json();
    return result.allowed;
  }

  async getResourceWithPermissions<T>(
    resourceId: string,
  ): Promise<ResourceWithPermissions<T>> {
    const response = await fetch(`/api/resources/${resourceId}`, {
      headers: {
        Authorization: `Bearer ${this.getToken()}`,
      },
    });

    if (response.status === 404) {
      throw new Error("Resource not found");
    }

    if (response.status === 403) {
      throw new Error("Access denied");
    }

    const data = await response.json();

    return {
      resource: data.resource,
      permissions: data.permissions, // What user can do with this resource
    };
  }

  private getToken(): string {
    // Get from secure storage
    return sessionStorage.getItem("auth_token") || "";
  }
}

type Action = "read" | "write" | "delete" | "share";

interface Permission {
  action: Action;
  allowed: boolean;
}

interface ResourceWithPermissions<T> {
  resource: T;
  permissions: Permission[];
}
```

## 2. Cryptographic Failures

Failures related to cryptography (or lack thereof) which often lead to exposure of sensitive data.

```typescript
// ❌ BAD: Storing sensitive data without encryption
localStorage.setItem("creditCard", "4532-1234-5678-9010");
localStorage.setItem("password", "myPassword123");

// ❌ BAD: Using weak encryption
function weakEncrypt(text: string): string {
  return btoa(text); // Base64 is NOT encryption!
}

// ✅ GOOD: Using Web Crypto API for encryption
class CryptoManager {
  private key: CryptoKey | null = null;

  async generateKey(): Promise<CryptoKey> {
    this.key = await crypto.subtle.generateKey(
      {
        name: "AES-GCM",
        length: 256,
      },
      true, // extractable
      ["encrypt", "decrypt"],
    );

    return this.key;
  }

  async encrypt(data: string): Promise<EncryptedData> {
    if (!this.key) {
      throw new Error("Key not initialized");
    }

    const encoder = new TextEncoder();
    const dataBuffer = encoder.encode(data);

    // Generate random IV for each encryption
    const iv = crypto.getRandomValues(new Uint8Array(12));

    const encryptedBuffer = await crypto.subtle.encrypt(
      {
        name: "AES-GCM",
        iv: iv,
      },
      this.key,
      dataBuffer,
    );

    return {
      ciphertext: Array.from(new Uint8Array(encryptedBuffer)),
      iv: Array.from(iv),
    };
  }

  async decrypt(encryptedData: EncryptedData): Promise<string> {
    if (!this.key) {
      throw new Error("Key not initialized");
    }

    const decryptedBuffer = await crypto.subtle.decrypt(
      {
        name: "AES-GCM",
        iv: new Uint8Array(encryptedData.iv),
      },
      this.key,
      new Uint8Array(encryptedData.ciphertext),
    );

    const decoder = new TextDecoder();
    return decoder.decode(decryptedBuffer);
  }

  // Hash passwords before sending to server
  async hashPassword(password: string, salt: string): Promise<string> {
    const encoder = new TextEncoder();
    const data = encoder.encode(password + salt);

    const hashBuffer = await crypto.subtle.digest("SHA-256", data);
    const hashArray = Array.from(new Uint8Array(hashBuffer));

    return hashArray.map((b) => b.toString(16).padStart(2, "0")).join("");
  }

  // Generate PBKDF2 key for password-based encryption
  async deriveKeyFromPassword(
    password: string,
    salt: Uint8Array,
  ): Promise<CryptoKey> {
    const encoder = new TextEncoder();
    const passwordBuffer = encoder.encode(password);

    const baseKey = await crypto.subtle.importKey(
      "raw",
      passwordBuffer,
      "PBKDF2",
      false,
      ["deriveKey"],
    );

    return await crypto.subtle.deriveKey(
      {
        name: "PBKDF2",
        salt: salt,
        iterations: 100000,
        hash: "SHA-256",
      },
      baseKey,
      {
        name: "AES-GCM",
        length: 256,
      },
      true,
      ["encrypt", "decrypt"],
    );
  }
}

interface EncryptedData {
  ciphertext: number[];
  iv: number[];
}
```

### Secure Data Transmission

```typescript
// Secure data transmission manager
class SecureTransmissionManager {
  private publicKey: CryptoKey | null = null;
  private privateKey: CryptoKey | null = null;

  async generateKeyPair(): Promise<void> {
    const keyPair = await crypto.subtle.generateKey(
      {
        name: "RSA-OAEP",
        modulusLength: 2048,
        publicExponent: new Uint8Array([1, 0, 1]),
        hash: "SHA-256",
      },
      true,
      ["encrypt", "decrypt"],
    );

    this.publicKey = keyPair.publicKey;
    this.privateKey = keyPair.privateKey;
  }

  async encryptForTransmission(data: string): Promise<string> {
    if (!this.publicKey) {
      throw new Error("Public key not initialized");
    }

    const encoder = new TextEncoder();
    const dataBuffer = encoder.encode(data);

    const encryptedBuffer = await crypto.subtle.encrypt(
      {
        name: "RSA-OAEP",
      },
      this.publicKey,
      dataBuffer,
    );

    const encryptedArray = Array.from(new Uint8Array(encryptedBuffer));
    return btoa(String.fromCharCode(...encryptedArray));
  }

  async decryptFromTransmission(encryptedData: string): Promise<string> {
    if (!this.privateKey) {
      throw new Error("Private key not initialized");
    }

    const encryptedArray = Uint8Array.from(atob(encryptedData), (c) =>
      c.charCodeAt(0),
    );

    const decryptedBuffer = await crypto.subtle.decrypt(
      {
        name: "RSA-OAEP",
      },
      this.privateKey,
      encryptedArray,
    );

    const decoder = new TextDecoder();
    return decoder.decode(decryptedBuffer);
  }

  // Export public key to share with server
  async exportPublicKey(): Promise<string> {
    if (!this.publicKey) {
      throw new Error("Public key not initialized");
    }

    const exported = await crypto.subtle.exportKey("spki", this.publicKey);
    const exportedArray = Array.from(new Uint8Array(exported));
    return btoa(String.fromCharCode(...exportedArray));
  }
}
```

## 3. Injection Attacks (XSS, Script Injection)

Injection flaws occur when untrusted data is sent to an interpreter as part of a command or query.

```typescript
// ❌ BAD: Direct HTML injection
function displayUserComment(comment: string) {
  document.getElementById("comment").innerHTML = comment; // XSS VULNERABILITY!
}

// ❌ BAD: eval() with user input
function calculateUserExpression(expression: string) {
  return eval(expression); // DANGEROUS!
}

// ✅ GOOD: Proper sanitization and escaping
class XSSProtection {
  // HTML entity encoding
  static escapeHTML(unsafe: string): string {
    return unsafe
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;")
      .replace(/"/g, "&quot;")
      .replace(/'/g, "&#039;");
  }

  // JavaScript string escaping
  static escapeJS(unsafe: string): string {
    return unsafe
      .replace(/\\/g, "\\\\")
      .replace(/'/g, "\\'")
      .replace(/"/g, '\\"')
      .replace(/\n/g, "\\n")
      .replace(/\r/g, "\\r")
      .replace(/\t/g, "\\t");
  }

  // URL encoding
  static escapeURL(unsafe: string): string {
    return encodeURIComponent(unsafe);
  }

  // Sanitize HTML with allowed tags
  static sanitizeHTML(dirty: string, allowedTags: string[] = []): string {
    const div = document.createElement("div");
    div.textContent = dirty;
    let clean = div.innerHTML;

    // Allow specific tags if needed
    allowedTags.forEach((tag) => {
      const regex = new RegExp(`&lt;${tag}&gt;`, "gi");
      clean = clean.replace(regex, `<${tag}>`);
      const closeRegex = new RegExp(`&lt;/${tag}&gt;`, "gi");
      clean = clean.replace(closeRegex, `</${tag}>`);
    });

    return clean;
  }

  // Create safe DOM element
  static createSafeElement(
    tag: string,
    content: string,
    attributes: Record<string, string> = {},
  ): HTMLElement {
    const element = document.createElement(tag);
    element.textContent = content; // Safe - uses text content, not HTML

    // Safely set attributes
    Object.entries(attributes).forEach(([key, value]) => {
      // Prevent javascript: URLs and event handlers
      if (
        key.toLowerCase().startsWith("on") ||
        value.startsWith("javascript:")
      ) {
        return;
      }
      element.setAttribute(key, value);
    });

    return element;
  }
}

// React-specific XSS protection
function SafeUserContent({ content }: { content: string }) {
  // React automatically escapes content in JSX
  return <div>{ content } < /div>; / / Safe;

  // If you MUST use dangerouslySetInnerHTML, sanitize first
  // return <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(content) }} />;
}
```

### Content Security Policy (CSP) Implementation

```typescript
// CSP Manager
class CSPManager {
  private policy: CSPDirectives;

  constructor() {
    this.policy = {
      "default-src": ["'self'"],
      "script-src": ["'self'", "'unsafe-inline'"], // Avoid unsafe-inline in production
      "style-src": ["'self'", "'unsafe-inline'"],
      "img-src": ["'self'", "data:", "https:"],
      "font-src": ["'self'"],
      "connect-src": ["'self'", "https://api.example.com"],
      "frame-ancestors": ["'none'"],
      "base-uri": ["'self'"],
      "form-action": ["'self'"],
    };
  }

  generateCSPHeader(): string {
    return Object.entries(this.policy)
      .map(([directive, sources]) => `${directive} ${sources.join(" ")}`)
      .join("; ");
  }

  setMetaTag(): void {
    const meta = document.createElement("meta");
    meta.httpEquiv = "Content-Security-Policy";
    meta.content = this.generateCSPHeader();
    document.head.appendChild(meta);
  }

  // Use nonce for inline scripts
  generateNonce(): string {
    const array = new Uint8Array(16);
    crypto.getRandomValues(array);
    return btoa(String.fromCharCode(...array));
  }

  addInlineScript(code: string, nonce: string): void {
    const script = document.createElement("script");
    script.textContent = code;
    script.nonce = nonce;
    document.body.appendChild(script);
  }
}

interface CSPDirectives {
  "default-src": string[];
  "script-src": string[];
  "style-src": string[];
  "img-src": string[];
  "font-src": string[];
  "connect-src": string[];
  "frame-ancestors": string[];
  "base-uri": string[];
  "form-action": string[];
}
```

## 4. Insecure Design

Risks related to design and architectural flaws with focus on missing or ineffective control design.

```typescript
// ❌ BAD: No rate limiting, no validation
async function submitForm(data: any) {
  await fetch("/api/submit", {
    method: "POST",
    body: JSON.stringify(data),
  });
}

// ✅ GOOD: Comprehensive secure design
class SecureFormHandler {
  private rateLimiter: RateLimiter;
  private validator: Validator;
  private csrf: CSRFProtection;

  constructor() {
    this.rateLimiter = new RateLimiter({
      maxRequests: 5,
      windowMs: 60000, // 1 minute
    });
    this.validator = new Validator();
    this.csrf = new CSRFProtection();
  }

  async submitForm<T>(data: T, schema: ValidationSchema): Promise<Response> {
    // 1. Rate limiting
    if (!this.rateLimiter.allow()) {
      throw new Error("Too many requests. Please try again later.");
    }

    // 2. Input validation
    const validationResult = this.validator.validate(data, schema);
    if (!validationResult.valid) {
      throw new Error(
        `Validation failed: ${validationResult.errors.join(", ")}`,
      );
    }

    // 3. CSRF protection
    const csrfToken = await this.csrf.getToken();

    // 4. Secure transmission
    const response = await fetch("/api/submit", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-CSRF-Token": csrfToken,
      },
      body: JSON.stringify(data),
      credentials: "same-origin", // Include cookies
    });

    // 5. Response validation
    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message || "Submission failed");
    }

    return response;
  }
}

// Rate Limiter Implementation
class RateLimiter {
  private requests: number[] = [];
  private maxRequests: number;
  private windowMs: number;

  constructor(config: { maxRequests: number; windowMs: number }) {
    this.maxRequests = config.maxRequests;
    this.windowMs = config.windowMs;
  }

  allow(): boolean {
    const now = Date.now();

    // Remove expired requests
    this.requests = this.requests.filter((time) => now - time < this.windowMs);

    if (this.requests.length >= this.maxRequests) {
      return false;
    }

    this.requests.push(now);
    return true;
  }

  getRemainingRequests(): number {
    return Math.max(0, this.maxRequests - this.requests.length);
  }
}

// Validator
class Validator {
  validate<T>(data: T, schema: ValidationSchema): ValidationResult {
    const errors: string[] = [];

    for (const [field, rules] of Object.entries(schema)) {
      const value = (data as any)[field];

      if (
        rules.required &&
        (value === undefined || value === null || value === "")
      ) {
        errors.push(`${field} is required`);
        continue;
      }

      if (value && rules.type && typeof value !== rules.type) {
        errors.push(`${field} must be of type ${rules.type}`);
      }

      if (value && rules.minLength && value.length < rules.minLength) {
        errors.push(`${field} must be at least ${rules.minLength} characters`);
      }

      if (value && rules.maxLength && value.length > rules.maxLength) {
        errors.push(`${field} must be at most ${rules.maxLength} characters`);
      }

      if (value && rules.pattern && !rules.pattern.test(value)) {
        errors.push(`${field} format is invalid`);
      }

      if (value && rules.custom) {
        const customError = rules.custom(value);
        if (customError) {
          errors.push(customError);
        }
      }
    }

    return {
      valid: errors.length === 0,
      errors,
    };
  }
}

interface ValidationSchema {
  [field: string]: {
    required?: boolean;
    type?: string;
    minLength?: number;
    maxLength?: number;
    pattern?: RegExp;
    custom?: (value: any) => string | null;
  };
}

interface ValidationResult {
  valid: boolean;
  errors: string[];
}
```

## 5. Security Misconfiguration

```typescript
// ❌ BAD: Exposing sensitive information
const config = {
  apiKey: "sk_live_1234567890abcdef", // Exposed in frontend!
  databaseUrl: "mongodb://user:pass@host:port/db", // Never in frontend!
  debug: true, // Should be false in production
};

// ❌ BAD: CORS misconfiguration
// Backend allowing all origins
res.setHeader("Access-Control-Allow-Origin", "*"); // DANGEROUS!

// ✅ GOOD: Proper configuration management
class ConfigManager {
  private config: AppConfig;

  constructor() {
    // Only public configuration in frontend
    this.config = {
      apiBaseUrl: import.meta.env.VITE_API_URL || "https://api.example.com",
      environment: import.meta.env.MODE || "production",
      publicKey: import.meta.env.VITE_PUBLIC_KEY || "",
      enableDebugMode: import.meta.env.MODE === "development",
      corsAllowedOrigins: [window.location.origin],
      maxFileUploadSize: 5 * 1024 * 1024, // 5MB
    };

    this.validateConfig();
  }

  private validateConfig(): void {
    if (!this.config.apiBaseUrl) {
      throw new Error("API base URL not configured");
    }

    if (
      !this.config.apiBaseUrl.startsWith("https://") &&
      this.config.environment === "production"
    ) {
      throw new Error("API must use HTTPS in production");
    }
  }

  getApiUrl(endpoint: string): string {
    return `${this.config.apiBaseUrl}${endpoint}`;
  }

  isDebugMode(): boolean {
    return this.config.enableDebugMode;
  }

  // Never log sensitive information
  safeLog(message: string, data?: any): void {
    if (!this.isDebugMode()) return;

    const sanitizedData = data ? this.sanitizeForLogging(data) : undefined;
    console.log(message, sanitizedData);
  }

  private sanitizeForLogging(data: any): any {
    const sensitive = ["password", "token", "apiKey", "secret", "creditCard"];

    if (typeof data !== "object" || data === null) return data;

    const sanitized: any = Array.isArray(data) ? [] : {};

    for (const [key, value] of Object.entries(data)) {
      if (sensitive.some((s) => key.toLowerCase().includes(s))) {
        sanitized[key] = "[REDACTED]";
      } else if (typeof value === "object" && value !== null) {
        sanitized[key] = this.sanitizeForLogging(value);
      } else {
        sanitized[key] = value;
      }
    }

    return sanitized;
  }
}

interface AppConfig {
  apiBaseUrl: string;
  environment: string;
  publicKey: string;
  enableDebugMode: boolean;
  corsAllowedOrigins: string[];
  maxFileUploadSize: number;
}
```

### Security Headers Implementation

```typescript
// Security Headers Manager
class SecurityHeadersManager {
  static readonly RECOMMENDED_HEADERS = {
    "Strict-Transport-Security": "max-age=31536000; includeSubDomains; preload",
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "X-XSS-Protection": "1; mode=block",
    "Referrer-Policy": "strict-origin-when-cross-origin",
    "Permissions-Policy": "geolocation=(), microphone=(), camera=()",
  };

  static validateResponseHeaders(response: Response): SecurityHeadersReport {
    const report: SecurityHeadersReport = {
      score: 100,
      missing: [],
      warnings: [],
    };

    for (const [header, expectedValue] of Object.entries(
      this.RECOMMENDED_HEADERS,
    )) {
      const actualValue = response.headers.get(header);

      if (!actualValue) {
        report.missing.push(header);
        report.score -= 15;
      } else if (actualValue !== expectedValue) {
        report.warnings.push(
          `${header}: Expected "${expectedValue}", got "${actualValue}"`,
        );
        report.score -= 5;
      }
    }

    return report;
  }

  static checkCSPViolations(): void {
    document.addEventListener(
      "securitypolicyviolation",
      (e: SecurityPolicyViolationEvent) => {
        console.error("CSP Violation:", {
          violatedDirective: e.violatedDirective,
          blockedURI: e.blockedURI,
          lineNumber: e.lineNumber,
          sourceFile: e.sourceFile,
        });

        // Report to monitoring service
        this.reportSecurityViolation({
          type: "csp_violation",
          directive: e.violatedDirective,
          uri: e.blockedURI,
        });
      },
    );
  }

  private static async reportSecurityViolation(violation: any): Promise<void> {
    try {
      await fetch("/api/security/report", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(violation),
      });
    } catch (error) {
      console.error("Failed to report security violation:", error);
    }
  }
}

interface SecurityHeadersReport {
  score: number;
  missing: string[];
  warnings: string[];
}
```

## 6. Vulnerable and Outdated Components

```typescript
// Package vulnerability scanner
class DependencySecurityScanner {
  private knownVulnerabilities: Map<string, Vulnerability[]> = new Map();

  async scanPackageJson(): Promise<SecurityReport> {
    const response = await fetch("/package.json");
    const packageData = await response.json();

    const report: SecurityReport = {
      totalDependencies: 0,
      vulnerableDependencies: [],
      recommendations: [],
    };

    const allDependencies = {
      ...packageData.dependencies,
      ...packageData.devDependencies,
    };

    for (const [name, version] of Object.entries(allDependencies)) {
      report.totalDependencies++;

      const vulns = await this.checkVulnerabilities(name, version as string);
      if (vulns.length > 0) {
        report.vulnerableDependencies.push({
          package: name,
          version: version as string,
          vulnerabilities: vulns,
        });
      }
    }

    this.generateRecommendations(report);
    return report;
  }

  private async checkVulnerabilities(
    packageName: string,
    version: string,
  ): Promise<Vulnerability[]> {
    // In production, use actual vulnerability database API
    // Example: npm audit, Snyk, GitHub Security Advisory
    const response = await fetch(
      `https://registry.npmjs.org/-/npm/v1/security/advisories/bulk`,
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ [packageName]: [version] }),
      },
    );

    const data = await response.json();
    return data[packageName] || [];
  }

  private generateRecommendations(report: SecurityReport): void {
    if (report.vulnerableDependencies.length > 0) {
      report.recommendations.push(
        "Run `npm audit fix` to automatically fix vulnerabilities",
      );
      report.recommendations.push("Review and update dependencies regularly");
      report.recommendations.push(
        "Consider using Dependabot or Renovate for automated updates",
      );
    }

    const highSeverity = report.vulnerableDependencies.filter((d) =>
      d.vulnerabilities.some(
        (v) => v.severity === "high" || v.severity === "critical",
      ),
    );

    if (highSeverity.length > 0) {
      report.recommendations.push(
        `URGENT: ${highSeverity.length} high/critical vulnerabilities found`,
      );
    }
  }
}

interface Vulnerability {
  id: string;
  title: string;
  severity: "low" | "moderate" | "high" | "critical";
  description: string;
  fixedIn: string;
}

interface SecurityReport {
  totalDependencies: number;
  vulnerableDependencies: Array<{
    package: string;
    version: string;
    vulnerabilities: Vulnerability[];
  }>;
  recommendations: string[];
}
```

## 7. Identification and Authentication Failures

```typescript
// ❌ BAD: Weak authentication
function login(username: string, password: string) {
  if (username === "admin" && password === "admin") {
    localStorage.setItem("user", username);
    return true;
  }
  return false;
}

// ✅ GOOD: Secure authentication system
class AuthenticationManager {
  private readonly MIN_PASSWORD_LENGTH = 12;
  private readonly MAX_LOGIN_ATTEMPTS = 5;
  private readonly LOCKOUT_DURATION = 15 * 60 * 1000; // 15 minutes
  private loginAttempts: Map<string, LoginAttempt[]> = new Map();

  async register(credentials: RegistrationCredentials): Promise<void> {
    // 1. Validate password strength
    const passwordStrength = this.checkPasswordStrength(credentials.password);
    if (passwordStrength.score < 3) {
      throw new Error(`Weak password: ${passwordStrength.feedback.join(", ")}`);
    }

    // 2. Check for common passwords
    if (await this.isCommonPassword(credentials.password)) {
      throw new Error(
        "Password is too common. Please choose a different password.",
      );
    }

    // 3. Validate email
    if (!this.isValidEmail(credentials.email)) {
      throw new Error("Invalid email address");
    }

    // 4. Send to server (password will be hashed server-side)
    const response = await fetch("/api/auth/register", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        email: credentials.email,
        password: credentials.password, // Sent over HTTPS
        mfaEnabled: true, // Enable MFA by default
      }),
    });

    if (!response.ok) {
      throw new Error("Registration failed");
    }
  }

  async login(
    email: string,
    password: string,
    mfaCode?: string,
  ): Promise<AuthResult> {
    // 1. Check if account is locked
    if (this.isAccountLocked(email)) {
      throw new Error(
        "Account temporarily locked due to too many failed attempts",
      );
    }

    // 2. Attempt login
    const response = await fetch("/api/auth/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, password, mfaCode }),
    });

    if (!response.ok) {
      this.recordFailedAttempt(email);

      const attemptsRemaining = this.getRemainingAttempts(email);
      throw new Error(`Login failed. ${attemptsRemaining} attempts remaining.`);
    }

    // 3. Successful login - reset attempts
    this.loginAttempts.delete(email);

    const result: AuthResult = await response.json();

    // 4. Store session securely
    await this.storeSession(result);

    return result;
  }

  private checkPasswordStrength(password: string): PasswordStrength {
    const feedback: string[] = [];
    let score = 0;

    // Length check
    if (password.length >= this.MIN_PASSWORD_LENGTH) {
      score++;
    } else {
      feedback.push(
        `Password must be at least ${this.MIN_PASSWORD_LENGTH} characters`,
      );
    }

    // Complexity checks
    if (/[a-z]/.test(password) && /[A-Z]/.test(password)) {
      score++;
    } else {
      feedback.push("Include both uppercase and lowercase letters");
    }

    if (/\d/.test(password)) {
      score++;
    } else {
      feedback.push("Include at least one number");
    }

    if (/[^a-zA-Z0-9]/.test(password)) {
      score++;
    } else {
      feedback.push("Include at least one special character");
    }

    // Check for common patterns
    if (!/(.)\1{2,}/.test(password)) {
      score++;
    } else {
      feedback.push("Avoid repeated characters");
    }

    return { score, feedback };
  }

  private async isCommonPassword(password: string): Promise<boolean> {
    // Check against a list of common passwords
    const commonPasswords = [
      "password",
      "123456",
      "qwerty",
      "admin",
      "letmein",
    ];
    return commonPasswords.includes(password.toLowerCase());
  }

  private isValidEmail(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }

  private isAccountLocked(email: string): boolean {
    const attempts = this.loginAttempts.get(email) || [];
    if (attempts.length < this.MAX_LOGIN_ATTEMPTS) {
      return false;
    }

    const lastAttempt = attempts[attempts.length - 1];
    return Date.now() - lastAttempt.timestamp < this.LOCKOUT_DURATION;
  }

  private recordFailedAttempt(email: string): void {
    const attempts = this.loginAttempts.get(email) || [];
    attempts.push({ timestamp: Date.now(), success: false });
    this.loginAttempts.set(email, attempts);
  }

  private getRemainingAttempts(email: string): number {
    const attempts = this.loginAttempts.get(email) || [];
    return Math.max(0, this.MAX_LOGIN_ATTEMPTS - attempts.length);
  }

  private async storeSession(authResult: AuthResult): Promise<void> {
    // Store in httpOnly cookie (set by server)
    // Or store access token in memory only
    sessionStorage.setItem("session_id", authResult.sessionId);
  }
}

interface RegistrationCredentials {
  email: string;
  password: string;
}

interface LoginAttempt {
  timestamp: number;
  success: boolean;
}

interface PasswordStrength {
  score: number;
  feedback: string[];
}

interface AuthResult {
  sessionId: string;
  user: {
    id: string;
    email: string;
  };
  requiresMFA: boolean;
}
```

### Multi-Factor Authentication (MFA)

```typescript
// MFA Implementation
class MFAManager {
  async setupTOTP(userId: string): Promise<TOTPSetup> {
    const response = await fetch("/api/auth/mfa/setup", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ userId }),
    });

    const data = await response.json();

    return {
      secret: data.secret,
      qrCode: data.qrCode,
      backupCodes: data.backupCodes,
    };
  }

  async verifyTOTP(userId: string, code: string): Promise<boolean> {
    const response = await fetch("/api/auth/mfa/verify", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ userId, code }),
    });

    const data = await response.json();
    return data.valid;
  }

  generateBackupCodes(count: number = 10): string[] {
    const codes: string[] = [];

    for (let i = 0; i < count; i++) {
      const array = new Uint8Array(8);
      crypto.getRandomValues(array);
      const code = Array.from(array, (byte) =>
        byte.toString(16).padStart(2, "0"),
      ).join("");
      codes.push(code);
    }

    return codes;
  }
}

interface TOTPSetup {
  secret: string;
  qrCode: string;
  backupCodes: string[];
}
```

## 8. Software and Data Integrity Failures

```typescript
// Subresource Integrity (SRI) Manager
class IntegrityManager {
  async generateSRI(url: string): Promise<string> {
    const response = await fetch(url);
    const content = await response.arrayBuffer();

    const hashBuffer = await crypto.subtle.digest("SHA-384", content);
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    const hashBase64 = btoa(String.fromCharCode(...hashArray));

    return `sha384-${hashBase64}`;
  }

  createScriptWithIntegrity(src: string, integrity: string): HTMLScriptElement {
    const script = document.createElement("script");
    script.src = src;
    script.integrity = integrity;
    script.crossOrigin = "anonymous";

    script.onerror = () => {
      console.error("Script integrity check failed:", src);
      this.reportIntegrityFailure(src);
    };

    return script;
  }

  verifySoftwareUpdate(update: SoftwareUpdate): boolean {
    // Verify digital signature of updates
    // This would typically use a public key to verify
    // the signature provided with the update
    return this.verifySignature(
      update.data,
      update.signature,
      update.publicKey,
    );
  }

  private verifySignature(
    data: string,
    signature: string,
    publicKey: string,
  ): boolean {
    // Implement signature verification
    // In production, use Web Crypto API or similar
    return true; // Placeholder
  }

  private async reportIntegrityFailure(resource: string): Promise<void> {
    await fetch("/api/security/integrity-failure", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        resource,
        timestamp: Date.now(),
        userAgent: navigator.userAgent,
      }),
    });
  }
}

interface SoftwareUpdate {
  version: string;
  data: string;
  signature: string;
  publicKey: string;
}
```

## 9. Security Logging and Monitoring

```typescript
// Security Event Logger
class SecurityLogger {
  private events: SecurityEvent[] = [];
  private readonly MAX_EVENTS = 1000;

  logSecurityEvent(event: Omit<SecurityEvent, "timestamp" | "id">): void {
    const securityEvent: SecurityEvent = {
      ...event,
      id: crypto.randomUUID(),
      timestamp: Date.now(),
    };

    this.events.push(securityEvent);

    // Keep only recent events
    if (this.events.length > this.MAX_EVENTS) {
      this.events.shift();
    }

    // Send critical events immediately
    if (event.severity === "critical" || event.severity === "high") {
      this.sendToServer(securityEvent);
    }

    // Log locally
    this.logToConsole(securityEvent);
  }

  private async sendToServer(event: SecurityEvent): Promise<void> {
    try {
      await fetch("/api/security/log", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(event),
        keepalive: true, // Ensure log is sent even if page unloads
      });
    } catch (error) {
      console.error("Failed to send security log:", error);
    }
  }

  private logToConsole(event: SecurityEvent): void {
    const message = `[SECURITY] ${event.type}: ${event.message}`;

    switch (event.severity) {
      case "critical":
      case "high":
        console.error(message, event.details);
        break;
      case "medium":
        console.warn(message, event.details);
        break;
      case "low":
        console.info(message, event.details);
        break;
    }
  }

  getEvents(filter?: EventFilter): SecurityEvent[] {
    if (!filter) return [...this.events];

    return this.events.filter((event) => {
      if (filter.type && event.type !== filter.type) return false;
      if (filter.severity && event.severity !== filter.severity) return false;
      if (filter.startTime && event.timestamp < filter.startTime) return false;
      if (filter.endTime && event.timestamp > filter.endTime) return false;
      return true;
    });
  }

  // Export events for analysis
  exportEvents(): string {
    return JSON.stringify(this.events, null, 2);
  }
}

interface SecurityEvent {
  id: string;
  type: SecurityEventType;
  severity: "low" | "medium" | "high" | "critical";
  message: string;
  details?: any;
  timestamp: number;
  userId?: string;
  ipAddress?: string;
}

type SecurityEventType =
  | "authentication_failed"
  | "authentication_success"
  | "authorization_failed"
  | "csrf_detected"
  | "xss_attempt"
  | "sql_injection_attempt"
  | "rate_limit_exceeded"
  | "suspicious_activity"
  | "data_breach_attempt"
  | "integrity_violation";

interface EventFilter {
  type?: SecurityEventType;
  severity?: string;
  startTime?: number;
  endTime?: number;
}

// Usage example
const logger = new SecurityLogger();

logger.logSecurityEvent({
  type: "authentication_failed",
  severity: "high",
  message: "Multiple failed login attempts detected",
  details: {
    attempts: 5,
    email: "user@example.com",
  },
  userId: "user123",
});
```

## 10. Server-Side Request Forgery (SSRF)

```typescript
// SSRF Protection
class SSRFProtection {
  private readonly ALLOWED_PROTOCOLS = ["http:", "https:"];
  private readonly BLOCKED_HOSTS = [
    "localhost",
    "127.0.0.1",
    "0.0.0.0",
    "169.254.169.254", // AWS metadata
    "::1",
  ];
  private readonly ALLOWED_DOMAINS: string[] = [];

  constructor(allowedDomains: string[]) {
    this.ALLOWED_DOMAINS = allowedDomains;
  }

  validateURL(url: string): ValidationResult {
    const errors: string[] = [];

    try {
      const parsed = new URL(url);

      // Check protocol
      if (!this.ALLOWED_PROTOCOLS.includes(parsed.protocol)) {
        errors.push(`Protocol ${parsed.protocol} is not allowed`);
      }

      // Check for blocked hosts
      if (this.isBlockedHost(parsed.hostname)) {
        errors.push(`Host ${parsed.hostname} is blocked`);
      }

      // Check if domain is in allowlist
      if (this.ALLOWED_DOMAINS.length > 0) {
        if (!this.isAllowedDomain(parsed.hostname)) {
          errors.push(`Domain ${parsed.hostname} is not in allowlist`);
        }
      }

      // Check for IP addresses in private ranges
      if (this.isPrivateIP(parsed.hostname)) {
        errors.push("Private IP addresses are not allowed");
      }
    } catch (error) {
      errors.push("Invalid URL format");
    }

    return {
      valid: errors.length === 0,
      errors,
    };
  }

  private isBlockedHost(hostname: string): boolean {
    return this.BLOCKED_HOSTS.some((blocked) =>
      hostname.toLowerCase().includes(blocked.toLowerCase()),
    );
  }

  private isAllowedDomain(hostname: string): boolean {
    return this.ALLOWED_DOMAINS.some((allowed) => hostname.endsWith(allowed));
  }

  private isPrivateIP(hostname: string): boolean {
    // Check for private IP ranges
    const privateRanges = [
      /^10\./,
      /^172\.(1[6-9]|2[0-9]|3[0-1])\./,
      /^192\.168\./,
      /^127\./,
    ];

    return privateRanges.some((range) => range.test(hostname));
  }

  async fetchWithValidation(
    url: string,
    options?: RequestInit,
  ): Promise<Response> {
    const validation = this.validateURL(url);

    if (!validation.valid) {
      throw new Error(`URL validation failed: ${validation.errors.join(", ")}`);
    }

    return fetch(url, {
      ...options,
      redirect: "manual", // Prevent automatic redirects
    });
  }
}
```

## Real-World Production Example

```typescript
// Comprehensive Security Implementation for React App
class SecurityManager {
  private xssProtection: XSSProtection;
  private csp: CSPManager;
  private auth: AuthenticationManager;
  private logger: SecurityLogger;
  private ssrfProtection: SSRFProtection;

  constructor() {
    this.xssProtection = new XSSProtection();
    this.csp = new CSPManager();
    this.auth = new AuthenticationManager();
    this.logger = new SecurityLogger();
    this.ssrfProtection = new SSRFProtection([
      "example.com",
      "api.example.com",
    ]);

    this.initialize();
  }

  private initialize(): void {
    // Set up CSP
    this.csp.setMetaTag();

    // Monitor CSP violations
    SecurityHeadersManager.checkCSPViolations();

    // Set up global error handler
    window.addEventListener("error", (event) => {
      this.logger.logSecurityEvent({
        type: "suspicious_activity",
        severity: "medium",
        message: "Unexpected error occurred",
        details: {
          message: event.message,
          filename: event.filename,
          lineno: event.lineno,
        },
      });
    });

    // Monitor network requests
    this.monitorNetworkRequests();
  }

  private monitorNetworkRequests(): void {
    const originalFetch = window.fetch;

    window.fetch = async (...args) => {
      const [resource, config] = args;
      const url = typeof resource === "string" ? resource : resource.url;

      // Validate URL
      const validation = this.ssrfProtection.validateURL(url);
      if (!validation.valid) {
        this.logger.logSecurityEvent({
          type: "suspicious_activity",
          severity: "high",
          message: "Blocked potentially malicious request",
          details: { url, errors: validation.errors },
        });
        throw new Error("Request blocked by security policy");
      }

      return originalFetch.apply(window, args);
    };
  }

  // Secure API call wrapper
  async secureApiCall<T>(
    endpoint: string,
    options: SecureRequestOptions = {},
  ): Promise<T> {
    const config = new ConfigManager();
    const fullUrl = config.getApiUrl(endpoint);

    // Add security headers
    const headers = new Headers(options.headers);
    headers.set("X-Request-ID", crypto.randomUUID());
    headers.set("X-Client-Version", "1.0.0");

    // Add CSRF token if available
    const csrfToken = sessionStorage.getItem("csrf_token");
    if (csrfToken) {
      headers.set("X-CSRF-Token", csrfToken);
    }

    try {
      const response = await fetch(fullUrl, {
        ...options,
        headers,
        credentials: "same-origin",
      });

      // Check security headers in response
      const headerReport =
        SecurityHeadersManager.validateResponseHeaders(response);
      if (headerReport.score < 50) {
        this.logger.logSecurityEvent({
          type: "suspicious_activity",
          severity: "medium",
          message: "Response has poor security headers",
          details: headerReport,
        });
      }

      if (!response.ok) {
        throw new Error(`API call failed: ${response.statusText}`);
      }

      return await response.json();
    } catch (error) {
      this.logger.logSecurityEvent({
        type: "suspicious_activity",
        severity: "medium",
        message: "API call failed",
        details: { endpoint, error: (error as Error).message },
      });
      throw error;
    }
  }
}

interface SecureRequestOptions extends RequestInit {
  timeout?: number;
  retries?: number;
}

// Initialize security manager globally
const securityManager = new SecurityManager();
export default securityManager;
```

## Best Practices

1. **Defense in Depth**: Implement multiple layers of security
2. **Least Privilege**: Grant minimum necessary permissions
3. **Fail Securely**: Default to denial, not access
4. **Never Trust Client Input**: Always validate and sanitize
5. **Use HTTPS Everywhere**: Encrypt all data in transit
6. **Keep Dependencies Updated**: Regular security audits
7. **Implement Logging**: Monitor and log security events
8. **Security Headers**: Use all recommended headers
9. **CSRF Protection**: Implement token-based CSRF protection
10. **Regular Security Testing**: Penetration testing and code reviews

## Key Takeaways

1. **Broken Access Control** is the #1 vulnerability - never trust client-side permission checks, always validate on server

2. **Cryptographic failures** expose sensitive data - use Web Crypto API for encryption, never store secrets in frontend

3. **XSS attacks** remain critical - sanitize all user input, implement CSP, use React's built-in XSS protection

4. **Insecure design** requires holistic approach - rate limiting, input validation, CSRF protection, secure defaults

5. **Security misconfiguration** is preventable - proper CORS, security headers, no sensitive data in environment variables

6. **Vulnerable components** need monitoring - regular dependency audits, automated security scanning, timely updates

7. **Authentication failures** are exploitable - strong passwords, MFA, account lockout, secure session management

8. **Integrity failures** compromise trust - use SRI for CDN resources, verify software updates, implement checksums

9. **Security logging** enables detection - comprehensive event logging, real-time monitoring, incident response

10. **SSRF protection** prevents internal access - URL validation, allowlists, block private IPs, monitor redirects
