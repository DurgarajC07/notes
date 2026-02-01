# Security Checklist

## Table of Contents

- [Introduction](#introduction)
- [Authentication Security](#authentication-security)
- [Authorization and Access Control](#authorization-and-access-control)
- [Data Protection](#data-protection)
- [Input Validation and Sanitization](#input-validation-and-sanitization)
- [XSS Prevention](#xss-prevention)
- [CSRF Protection](#csrf-protection)
- [Content Security Policy](#content-security-policy)
- [Secure Communication](#secure-communication)
- [Dependency Security](#dependency-security)
- [Security Headers](#security-headers)
- [Secrets Management](#secrets-management)
- [Monitoring and Incident Response](#monitoring-and-incident-response)
- [Key Takeaways](#key-takeaways)

## Introduction

Security is not optional - it's fundamental. A single vulnerability can compromise user data, damage reputation, and result in legal consequences. This checklist ensures comprehensive security coverage.

### Security Principles

```typescript
interface SecurityPrinciples {
  defense_in_depth: "Multiple layers of security controls";
  least_privilege: "Minimum permissions necessary";
  fail_secure: "Default deny, explicit allow";
  zero_trust: "Never trust, always verify";
  security_by_design: "Build security in from start";
}

const securityMindset = {
  assume_breach: "Plan for compromise, not if but when",
  validate_input: "Never trust user input",
  encrypt_data: "Protect data at rest and in transit",
  audit_everything: "Log security-relevant events",
  update_regularly: "Keep dependencies current",
};
```

## Authentication Security

### ✅ Secure Authentication Implementation

```typescript
// auth-service.ts - Secure authentication
import { hash, compare } from "bcrypt";
import { sign, verify } from "jsonwebtoken";
import { randomBytes } from "crypto";

const SALT_ROUNDS = 12; // Bcrypt cost factor
const TOKEN_EXPIRY = "15m"; // Short-lived access tokens
const REFRESH_TOKEN_EXPIRY = "7d";

interface AuthTokens {
  accessToken: string;
  refreshToken: string;
}

class AuthService {
  // Hash password with bcrypt
  async hashPassword(password: string): Promise<string> {
    // Validate password strength first
    this.validatePasswordStrength(password);
    return hash(password, SALT_ROUNDS);
  }

  // Verify password
  async verifyPassword(password: string, hash: string): Promise<boolean> {
    // Constant-time comparison
    return compare(password, hash);
  }

  // Generate tokens
  async generateTokens(userId: string): Promise<AuthTokens> {
    const accessToken = sign(
      { userId, type: "access" },
      process.env.JWT_SECRET!,
      { expiresIn: TOKEN_EXPIRY, algorithm: "HS256" },
    );

    const refreshToken = sign(
      { userId, type: "refresh", jti: randomBytes(16).toString("hex") },
      process.env.JWT_REFRESH_SECRET!,
      { expiresIn: REFRESH_TOKEN_EXPIRY, algorithm: "HS256" },
    );

    // Store refresh token hash in database
    await this.storeRefreshToken(userId, refreshToken);

    return { accessToken, refreshToken };
  }

  // Verify access token
  verifyAccessToken(token: string): { userId: string } | null {
    try {
      const payload = verify(token, process.env.JWT_SECRET!) as any;

      if (payload.type !== "access") {
        throw new Error("Invalid token type");
      }

      return { userId: payload.userId };
    } catch (error) {
      return null;
    }
  }

  // Password strength validation
  private validatePasswordStrength(password: string): void {
    const minLength = 12;
    const requirements = {
      lowercase: /[a-z]/,
      uppercase: /[A-Z]/,
      number: /[0-9]/,
      special: /[^A-Za-z0-9]/,
    };

    if (password.length < minLength) {
      throw new Error(`Password must be at least ${minLength} characters`);
    }

    const metRequirements = Object.entries(requirements).filter(([_, regex]) =>
      regex.test(password),
    ).length;

    if (metRequirements < 3) {
      throw new Error("Password must meet complexity requirements");
    }

    // Check against common passwords
    if (this.isCommonPassword(password)) {
      throw new Error("Password is too common");
    }
  }

  private isCommonPassword(password: string): boolean {
    // Check against list of common passwords
    const commonPasswords = ["Password123!", "123456", "qwerty"];
    return commonPasswords.includes(password);
  }

  private async storeRefreshToken(
    userId: string,
    token: string,
  ): Promise<void> {
    const tokenHash = await hash(token, SALT_ROUNDS);
    // Store in database with expiry
    // await db.refreshTokens.create({ userId, tokenHash, expiresAt });
  }
}

// Multi-factor authentication
class MFAService {
  async generateTOTPSecret(userId: string): Promise<string> {
    const secret = randomBytes(32).toString("base64");
    // Store encrypted in database
    await this.storeTOTPSecret(userId, secret);
    return secret;
  }

  async verifyTOTP(userId: string, token: string): Promise<boolean> {
    const secret = await this.getTOTPSecret(userId);
    // Verify TOTP token against secret
    // Implementation using speakeasy or similar
    return true;
  }

  private async storeTOTPSecret(userId: string, secret: string): Promise<void> {
    // Encrypt secret before storing
    const encrypted = this.encrypt(secret);
    // await db.mfaSecrets.create({ userId, secret: encrypted });
  }

  private encrypt(data: string): string {
    // Implement encryption using crypto module
    return data; // Placeholder
  }

  private async getTOTPSecret(userId: string): Promise<string> {
    // Retrieve and decrypt
    return ""; // Placeholder
  }
}
```

### ✅ Secure Session Management

```typescript
// session-manager.ts
interface SessionConfig {
  name: string;
  secret: string;
  secure: boolean;
  httpOnly: boolean;
  sameSite: "strict" | "lax" | "none";
  maxAge: number;
  rolling: boolean;
}

const sessionConfig: SessionConfig = {
  name: "sessionId",
  secret: process.env.SESSION_SECRET!,
  secure: true, // HTTPS only
  httpOnly: true, // No JavaScript access
  sameSite: "strict", // CSRF protection
  maxAge: 24 * 60 * 60 * 1000, // 24 hours
  rolling: true, // Extend on activity
};

class SessionManager {
  private sessions = new Map<string, SessionData>();
  private readonly sessionTimeout = 30 * 60 * 1000; // 30 minutes

  async createSession(userId: string): Promise<string> {
    const sessionId = randomBytes(32).toString("hex");

    const session: SessionData = {
      userId,
      createdAt: Date.now(),
      lastActivity: Date.now(),
      ip: this.getClientIP(),
      userAgent: this.getUserAgent(),
    };

    this.sessions.set(sessionId, session);
    this.scheduleCleanup(sessionId);

    return sessionId;
  }

  async validateSession(sessionId: string): Promise<SessionData | null> {
    const session = this.sessions.get(sessionId);

    if (!session) return null;

    // Check timeout
    if (Date.now() - session.lastActivity > this.sessionTimeout) {
      this.sessions.delete(sessionId);
      return null;
    }

    // Verify IP and user agent (optional, can be too strict)
    if (this.shouldVerifyFingerprint()) {
      if (
        session.ip !== this.getClientIP() ||
        session.userAgent !== this.getUserAgent()
      ) {
        this.sessions.delete(sessionId);
        await this.notifySecurityEvent("session_hijacking_attempt");
        return null;
      }
    }

    // Update last activity
    session.lastActivity = Date.now();
    return session;
  }

  async invalidateSession(sessionId: string): Promise<void> {
    this.sessions.delete(sessionId);
  }

  async invalidateAllUserSessions(userId: string): Promise<void> {
    for (const [sessionId, session] of this.sessions) {
      if (session.userId === userId) {
        this.sessions.delete(sessionId);
      }
    }
  }

  private scheduleCleanup(sessionId: string): void {
    setTimeout(() => {
      this.sessions.delete(sessionId);
    }, this.sessionTimeout);
  }

  private getClientIP(): string {
    // Get from request headers
    return "0.0.0.0";
  }

  private getUserAgent(): string {
    // Get from request headers
    return "";
  }

  private shouldVerifyFingerprint(): boolean {
    return process.env.NODE_ENV === "production";
  }

  private async notifySecurityEvent(event: string): Promise<void> {
    // Log security event
    console.warn(`Security event: ${event}`);
  }
}

interface SessionData {
  userId: string;
  createdAt: number;
  lastActivity: number;
  ip: string;
  userAgent: string;
}
```

### ✅ OAuth and Social Login Security

```typescript
// oauth-service.ts
interface OAuthConfig {
  clientId: string;
  clientSecret: string;
  redirectUri: string;
  scope: string[];
}

class OAuthService {
  private pendingStates = new Map<string, { createdAt: number }>();

  async initiateOAuthFlow(provider: string): Promise<string> {
    // Generate CSRF state token
    const state = randomBytes(32).toString("hex");

    // Store state with timestamp (expires in 10 minutes)
    this.pendingStates.set(state, { createdAt: Date.now() });

    // Cleanup expired states
    this.cleanupExpiredStates();

    const config = this.getProviderConfig(provider);

    // Build authorization URL
    const authUrl = new URL(config.authorizationEndpoint);
    authUrl.searchParams.set("client_id", config.clientId);
    authUrl.searchParams.set("redirect_uri", config.redirectUri);
    authUrl.searchParams.set("response_type", "code");
    authUrl.searchParams.set("scope", config.scope.join(" "));
    authUrl.searchParams.set("state", state);

    return authUrl.toString();
  }

  async handleCallback(code: string, state: string): Promise<string> {
    // Verify state token
    if (!this.pendingStates.has(state)) {
      throw new Error("Invalid or expired state token");
    }

    this.pendingStates.delete(state);

    // Exchange code for tokens
    const tokens = await this.exchangeCodeForTokens(code);

    // Verify tokens
    const userInfo = await this.verifyAndGetUserInfo(tokens.idToken);

    // Create or update user
    const userId = await this.createOrUpdateUser(userInfo);

    return userId;
  }

  private cleanupExpiredStates(): void {
    const now = Date.now();
    const tenMinutes = 10 * 60 * 1000;

    for (const [state, data] of this.pendingStates) {
      if (now - data.createdAt > tenMinutes) {
        this.pendingStates.delete(state);
      }
    }
  }

  private getProviderConfig(provider: string): any {
    // Return provider-specific configuration
    return {};
  }

  private async exchangeCodeForTokens(code: string): Promise<any> {
    // Exchange authorization code for tokens
    return {};
  }

  private async verifyAndGetUserInfo(idToken: string): Promise<any> {
    // Verify JWT signature and extract user info
    return {};
  }

  private async createOrUpdateUser(userInfo: any): Promise<string> {
    // Create or update user in database
    return "";
  }
}
```

## Authorization and Access Control

### ✅ Role-Based Access Control (RBAC)

```typescript
// rbac.ts
enum Permission {
  READ_USER = 'read:user',
  WRITE_USER = 'write:user',
  DELETE_USER = 'delete:user',
  READ_ADMIN = 'read:admin',
  WRITE_ADMIN = 'write:admin'
}

enum Role {
  USER = 'user',
  MODERATOR = 'moderator',
  ADMIN = 'admin',
  SUPER_ADMIN = 'super_admin'
}

const rolePermissions: Record<Role, Permission[]> = {
  [Role.USER]: [
    Permission.READ_USER
  ],
  [Role.MODERATOR]: [
    Permission.READ_USER,
    Permission.WRITE_USER
  ],
  [Role.ADMIN]: [
    Permission.READ_USER,
    Permission.WRITE_USER,
    Permission.DELETE_USER,
    Permission.READ_ADMIN
  ],
  [Role.SUPER_ADMIN]: Object.values(Permission)
};

class RBACService {
  hasPermission(role: Role, permission: Permission): boolean {
    const permissions = rolePermissions[role] || [];
    return permissions.includes(permission);
  }

  hasAnyPermission(role: Role, permissions: Permission[]): boolean {
    return permissions.some(p => this.hasPermission(role, p));
  }

  hasAllPermissions(role: Role, permissions: Permission[]): boolean {
    return permissions.every(p => this.hasPermission(role, p));
  }

  async authorizeRequest(
    userId: string,
    resource: string,
    action: string
  ): Promise<boolean> {
    const user = await this.getUser(userId);
    const permission = this.getRequiredPermission(resource, action);

    return this.hasPermission(user.role, permission);
  }

  private async getUser(userId: string): Promise<{ role: Role }> {
    // Fetch user from database
    return { role: Role.USER };
  }

  private getRequiredPermission(resource: string, action: string): Permission {
    // Map resource and action to permission
    return Permission.READ_USER;
  }
}

// React authorization hook
function useAuthorization() {
  const user = useAuth();

  const can = useCallback((permission: Permission): boolean => {
    if (!user) return false;
    const rbac = new RBACService();
    return rbac.hasPermission(user.role, permission);
  }, [user]);

  return { can };
}

// Protected component
function AdminPanel() {
  const { can } = useAuthorization();

  if (!can(Permission.READ_ADMIN)) {
    return <AccessDenied />;
  }

  return <div>Admin content</div>;
}

// Protected route
function ProtectedRoute({
  element,
  permission
}: {
  element: React.ReactElement;
  permission: Permission;
}) {
  const { can } = useAuthorization();

  if (!can(permission)) {
    return <Navigate to="/unauthorized" />;
  }

  return element;
}
```

### ✅ Attribute-Based Access Control (ABAC)

```typescript
// abac.ts - More granular than RBAC
interface AccessContext {
  user: {
    id: string;
    role: Role;
    department: string;
  };
  resource: {
    type: string;
    ownerId?: string;
    visibility?: "public" | "private" | "restricted";
  };
  action: string;
  environment: {
    time: Date;
    ip: string;
  };
}

type PolicyRule = (context: AccessContext) => boolean;

class ABACService {
  private policies: Map<string, PolicyRule> = new Map();

  constructor() {
    this.registerDefaultPolicies();
  }

  authorize(context: AccessContext): boolean {
    const policyKey = `${context.resource.type}:${context.action}`;
    const policy = this.policies.get(policyKey);

    if (!policy) {
      // Default deny
      return false;
    }

    return policy(context);
  }

  private registerDefaultPolicies(): void {
    // Users can read public resources
    this.policies.set("document:read", (ctx) => {
      return (
        ctx.resource.visibility === "public" ||
        ctx.resource.ownerId === ctx.user.id ||
        ctx.user.role === Role.ADMIN
      );
    });

    // Users can edit their own resources
    this.policies.set("document:edit", (ctx) => {
      return (
        ctx.resource.ownerId === ctx.user.id || ctx.user.role === Role.ADMIN
      );
    });

    // Users can delete only their own resources
    this.policies.set("document:delete", (ctx) => {
      return (
        (ctx.resource.ownerId === ctx.user.id && ctx.user.role !== Role.USER) ||
        ctx.user.role === Role.ADMIN
      );
    });

    // Time-based access
    this.policies.set("report:read", (ctx) => {
      const hour = ctx.environment.time.getHours();
      return hour >= 9 && hour <= 17; // Business hours only
    });
  }
}
```

## Data Protection

### ✅ Encryption

```typescript
// encryption-service.ts
import { createCipheriv, createDecipheriv, randomBytes, scrypt } from "crypto";
import { promisify } from "util";

const scryptAsync = promisify(scrypt);

class EncryptionService {
  private algorithm = "aes-256-gcm";
  private keyLength = 32;
  private ivLength = 16;
  private saltLength = 32;
  private tagLength = 16;

  async encrypt(plaintext: string, password: string): Promise<string> {
    // Generate salt and IV
    const salt = randomBytes(this.saltLength);
    const iv = randomBytes(this.ivLength);

    // Derive key from password
    const key = await this.deriveKey(password, salt);

    // Encrypt
    const cipher = createCipheriv(this.algorithm, key, iv);
    const encrypted = Buffer.concat([
      cipher.update(plaintext, "utf8"),
      cipher.final(),
    ]);

    // Get auth tag
    const tag = cipher.getAuthTag();

    // Combine salt, iv, tag, and encrypted data
    const result = Buffer.concat([salt, iv, tag, encrypted]);

    return result.toString("base64");
  }

  async decrypt(ciphertext: string, password: string): Promise<string> {
    const data = Buffer.from(ciphertext, "base64");

    // Extract components
    const salt = data.subarray(0, this.saltLength);
    const iv = data.subarray(this.saltLength, this.saltLength + this.ivLength);
    const tag = data.subarray(
      this.saltLength + this.ivLength,
      this.saltLength + this.ivLength + this.tagLength,
    );
    const encrypted = data.subarray(
      this.saltLength + this.ivLength + this.tagLength,
    );

    // Derive key
    const key = await this.deriveKey(password, salt);

    // Decrypt
    const decipher = createDecipheriv(this.algorithm, key, iv);
    decipher.setAuthTag(tag);

    const decrypted = Buffer.concat([
      decipher.update(encrypted),
      decipher.final(),
    ]);

    return decrypted.toString("utf8");
  }

  private async deriveKey(password: string, salt: Buffer): Promise<Buffer> {
    return (await scryptAsync(password, salt, this.keyLength)) as Buffer;
  }

  // Hash sensitive data (one-way)
  async hashData(data: string): Promise<string> {
    const salt = randomBytes(16);
    const key = (await scryptAsync(data, salt, 64)) as Buffer;

    return `${salt.toString("hex")}:${key.toString("hex")}`;
  }

  async verifyHash(data: string, hash: string): Promise<boolean> {
    const [saltHex, keyHex] = hash.split(":");
    const salt = Buffer.from(saltHex, "hex");
    const originalKey = Buffer.from(keyHex, "hex");

    const derivedKey = (await scryptAsync(data, salt, 64)) as Buffer;

    // Constant-time comparison
    return crypto.timingSafeEqual(originalKey, derivedKey);
  }
}

// Usage in application
class SecureStorage {
  private encryption = new EncryptionService();

  async storeSecureData(userId: string, data: any): Promise<void> {
    const password = process.env.ENCRYPTION_KEY!;
    const encrypted = await this.encryption.encrypt(
      JSON.stringify(data),
      password,
    );

    // Store encrypted data
    // await db.secureData.create({ userId, data: encrypted });
  }

  async retrieveSecureData(userId: string): Promise<any> {
    // Retrieve encrypted data
    // const record = await db.secureData.findOne({ userId });
    const encrypted = ""; // Placeholder

    const password = process.env.ENCRYPTION_KEY!;
    const decrypted = await this.encryption.decrypt(encrypted, password);

    return JSON.parse(decrypted);
  }
}
```

### ✅ Secure Data Storage

```typescript
// secure-storage-client.ts - Browser storage security
class SecureLocalStorage {
  private encryptionKey: string;

  constructor() {
    // Derive key from user session or other secure source
    this.encryptionKey = this.getEncryptionKey();
  }

  async setItem(key: string, value: any): Promise<void> {
    const serialized = JSON.stringify(value);
    const encrypted = await this.encrypt(serialized);
    localStorage.setItem(key, encrypted);
  }

  async getItem<T>(key: string): Promise<T | null> {
    const encrypted = localStorage.getItem(key);
    if (!encrypted) return null;

    try {
      const decrypted = await this.decrypt(encrypted);
      return JSON.parse(decrypted);
    } catch {
      return null;
    }
  }

  removeItem(key: string): void {
    localStorage.removeItem(key);
  }

  private async encrypt(data: string): Promise<string> {
    // Use Web Crypto API
    const encoder = new TextEncoder();
    const dataBuffer = encoder.encode(data);

    const keyBuffer = await crypto.subtle.digest(
      "SHA-256",
      encoder.encode(this.encryptionKey),
    );

    const key = await crypto.subtle.importKey(
      "raw",
      keyBuffer,
      "AES-GCM",
      false,
      ["encrypt"],
    );

    const iv = crypto.getRandomValues(new Uint8Array(12));
    const encrypted = await crypto.subtle.encrypt(
      { name: "AES-GCM", iv },
      key,
      dataBuffer,
    );

    // Combine IV and encrypted data
    const combined = new Uint8Array(iv.length + encrypted.byteLength);
    combined.set(iv);
    combined.set(new Uint8Array(encrypted), iv.length);

    return btoa(String.fromCharCode(...combined));
  }

  private async decrypt(data: string): Promise<string> {
    const combined = Uint8Array.from(atob(data), (c) => c.charCodeAt(0));
    const iv = combined.slice(0, 12);
    const encrypted = combined.slice(12);

    const encoder = new TextEncoder();
    const keyBuffer = await crypto.subtle.digest(
      "SHA-256",
      encoder.encode(this.encryptionKey),
    );

    const key = await crypto.subtle.importKey(
      "raw",
      keyBuffer,
      "AES-GCM",
      false,
      ["decrypt"],
    );

    const decrypted = await crypto.subtle.decrypt(
      { name: "AES-GCM", iv },
      key,
      encrypted,
    );

    const decoder = new TextDecoder();
    return decoder.decode(decrypted);
  }

  private getEncryptionKey(): string {
    // Derive from session or other secure source
    // Never hardcode!
    return sessionStorage.getItem("_ek") || "";
  }
}

// Never store sensitive data in localStorage without encryption
const secureStorage = new SecureLocalStorage();

// Bad
localStorage.setItem("creditCard", "4111-1111-1111-1111");

// Good
await secureStorage.setItem("creditCard", "4111-1111-1111-1111");
```

## Input Validation and Sanitization

### ✅ Comprehensive Input Validation

```typescript
// validation.ts
import { z } from "zod";
import DOMPurify from "isomorphic-dompurify";

// Schema-based validation
const userSchema = z.object({
  email: z.string().email().max(255),
  username: z
    .string()
    .min(3)
    .max(30)
    .regex(/^[a-zA-Z0-9_]+$/),
  password: z.string().min(12).max(128),
  age: z.number().int().min(13).max(120),
  website: z.string().url().optional(),
  bio: z.string().max(500).optional(),
});

class ValidationService {
  // Validate user input
  validateUserInput(data: unknown): z.infer<typeof userSchema> {
    try {
      return userSchema.parse(data);
    } catch (error) {
      if (error instanceof z.ZodError) {
        throw new Error(`Validation failed: ${this.formatZodError(error)}`);
      }
      throw error;
    }
  }

  // Sanitize HTML input
  sanitizeHTML(html: string): string {
    return DOMPurify.sanitize(html, {
      ALLOWED_TAGS: ["b", "i", "em", "strong", "a", "p", "br"],
      ALLOWED_ATTR: ["href"],
      ALLOW_DATA_ATTR: false,
    });
  }

  // Sanitize SQL-like strings (still use parameterized queries!)
  escapeSQLString(input: string): string {
    return input.replace(/['";\\]/g, "\\$&");
  }

  // Validate file uploads
  validateFileUpload(file: File): void {
    const maxSize = 5 * 1024 * 1024; // 5MB
    const allowedTypes = ["image/jpeg", "image/png", "image/webp"];
    const allowedExtensions = [".jpg", ".jpeg", ".png", ".webp"];

    // Check size
    if (file.size > maxSize) {
      throw new Error("File too large");
    }

    // Check MIME type
    if (!allowedTypes.includes(file.type)) {
      throw new Error("Invalid file type");
    }

    // Check extension
    const extension = file.name.toLowerCase().match(/\.[^.]*$/)?.[0];
    if (!extension || !allowedExtensions.includes(extension)) {
      throw new Error("Invalid file extension");
    }

    // Additional: Check magic bytes for real file type
    // This prevents MIME type spoofing
  }

  // Validate URLs
  validateURL(url: string): boolean {
    try {
      const parsed = new URL(url);

      // Only allow HTTP(S)
      if (!["http:", "https:"].includes(parsed.protocol)) {
        return false;
      }

      // Prevent SSRF - block private IPs
      const hostname = parsed.hostname;
      if (this.isPrivateIP(hostname)) {
        return false;
      }

      return true;
    } catch {
      return false;
    }
  }

  private isPrivateIP(hostname: string): boolean {
    const privateRanges = [
      /^10\./,
      /^172\.(1[6-9]|2[0-9]|3[01])\./,
      /^192\.168\./,
      /^127\./,
      /^localhost$/i,
    ];

    return privateRanges.some((pattern) => pattern.test(hostname));
  }

  private formatZodError(error: z.ZodError): string {
    return error.errors
      .map((e) => `${e.path.join(".")}: ${e.message}`)
      .join(", ");
  }
}

// React form validation hook
function useValidation<T extends z.ZodType>(schema: T) {
  const [errors, setErrors] = useState<Record<string, string>>({});

  const validate = useCallback(
    (data: unknown): z.infer<T> | null => {
      try {
        const validated = schema.parse(data);
        setErrors({});
        return validated;
      } catch (error) {
        if (error instanceof z.ZodError) {
          const newErrors: Record<string, string> = {};
          error.errors.forEach((err) => {
            const path = err.path.join(".");
            newErrors[path] = err.message;
          });
          setErrors(newErrors);
        }
        return null;
      }
    },
    [schema],
  );

  return { validate, errors };
}
```

## XSS Prevention

### ✅ XSS Protection Strategies

```typescript
// xss-prevention.ts

// 1. Always escape output
function escapeHTML(str: string): string {
  const div = document.createElement('div');
  div.textContent = str;
  return div.innerHTML;
}

// 2. Use textContent instead of innerHTML
function safeSetText(element: HTMLElement, text: string): void {
  element.textContent = text; // Safe
  // element.innerHTML = text; // Dangerous!
}

// 3. Sanitize user-generated HTML
import DOMPurify from 'isomorphic-dompurify';

function SafeHTML({ html }: { html: string }) {
  const sanitized = useMemo(() => {
    return DOMPurify.sanitize(html, {
      ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'ol', 'li'],
      ALLOWED_ATTR: ['href', 'title'],
      ALLOW_DATA_ATTR: false,
      FORBID_TAGS: ['script', 'style', 'iframe', 'object', 'embed'],
      FORBID_ATTR: ['onerror', 'onload', 'onclick']
    });
  }, [html]);

  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}

// 4. Safe URL handling
function SafeLink({ href, children }: { href: string; children: React.ReactNode }) {
  const safeHref = useMemo(() => {
    try {
      const url = new URL(href);
      // Only allow http(s) protocols
      if (!['http:', 'https:'].includes(url.protocol)) {
        return '#';
      }
      return href;
    } catch {
      return '#';
    }
  }, [href]);

  return (
    <a
      href={safeHref}
      rel="noopener noreferrer"
      target="_blank"
    >
      {children}
    </a>
  );
}

// 5. Prevent DOM-based XSS
function SearchComponent() {
  const [query, setQuery] = useState('');
  const searchParams = useSearchParams();

  useEffect(() => {
    // Dangerous: Using URL parameter directly
    // const q = searchParams.get('q');
    // document.getElementById('result')!.innerHTML = q;

    // Safe: Sanitize first
    const q = searchParams.get('q');
    if (q) {
      const sanitized = DOMPurify.sanitize(q);
      document.getElementById('result')!.textContent = sanitized;
    }
  }, [searchParams]);

  return <div id="result"></div>;
}

// 6. Content Security Policy Meta Tag
function CSPMeta() {
  return (
    <meta
      httpEquiv="Content-Security-Policy"
      content="
        default-src 'self';
        script-src 'self' 'unsafe-inline';
        style-src 'self' 'unsafe-inline';
        img-src 'self' data: https:;
      "
    />
  );
}
```

## CSRF Protection

### ✅ CSRF Token Implementation

```typescript
// csrf-protection.ts

// Server-side: Generate CSRF token
class CSRFProtection {
  private tokens = new Map<string, { token: string; expires: number }>();

  generateToken(sessionId: string): string {
    const token = randomBytes(32).toString('hex');
    const expires = Date.now() + 60 * 60 * 1000; // 1 hour

    this.tokens.set(sessionId, { token, expires });

    return token;
  }

  validateToken(sessionId: string, token: string): boolean {
    const stored = this.tokens.get(sessionId);

    if (!stored) return false;
    if (stored.expires < Date.now()) {
      this.tokens.delete(sessionId);
      return false;
    }

    return stored.token === token;
  }
}

// Client-side: Include CSRF token in requests
class APIClient {
  private csrfToken: string | null = null;

  async setCSRFToken(): Promise<void> {
    const response = await fetch('/api/csrf-token');
    const { token } = await response.json();
    this.csrfToken = token;
  }

  async request(url: string, options: RequestInit = {}): Promise<Response> {
    const headers = new Headers(options.headers);

    // Add CSRF token to requests
    if (this.csrfToken && ['POST', 'PUT', 'DELETE', 'PATCH'].includes(options.method || '')) {
      headers.set('X-CSRF-Token', this.csrfToken);
    }

    return fetch(url, {
      ...options,
      headers,
      credentials: 'include' // Include cookies
    });
  }
}

// React hook for CSRF protection
function useCSRF() {
  const [csrfToken, setCSRFToken] = useState<string>('');

  useEffect(() => {
    fetch('/api/csrf-token', { credentials: 'include' })
      .then(res => res.json())
      .then(data => setCSRFToken(data.token));
  }, []);

  const makeRequest = useCallback(async (
    url: string,
    options: RequestInit = {}
  ): Promise<Response> => {
    const headers = new Headers(options.headers);
    headers.set('X-CSRF-Token', csrfToken);

    return fetch(url, {
      ...options,
      headers,
      credentials: 'include'
    });
  }, [csrfToken]);

  return { csrfToken, makeRequest };
}

// Form with CSRF protection
function SecureForm() {
  const { csrfToken, makeRequest } = useCSRF();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    const formData = new FormData(e.target as HTMLFormElement);

    await makeRequest('/api/submit', {
      method: 'POST',
      body: formData
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="hidden" name="csrf_token" value={csrfToken} />
      {/* form fields */}
      <button type="submit">Submit</button>
    </form>
  );
}
```

## Content Security Policy

### ✅ Comprehensive CSP Configuration

```typescript
// csp-config.ts
interface CSPDirectives {
  "default-src": string[];
  "script-src": string[];
  "style-src": string[];
  "img-src": string[];
  "font-src": string[];
  "connect-src": string[];
  "frame-src": string[];
  "object-src": string[];
  "base-uri": string[];
  "form-action": string[];
  "frame-ancestors": string[];
  "upgrade-insecure-requests"?: boolean;
}

const cspConfig: CSPDirectives = {
  "default-src": ["'self'"],
  "script-src": [
    "'self'",
    "'unsafe-inline'", // Avoid if possible
    "https://cdn.example.com",
  ],
  "style-src": [
    "'self'",
    "'unsafe-inline'", // Required for many CSS-in-JS solutions
    "https://fonts.googleapis.com",
  ],
  "img-src": ["'self'", "data:", "https:", "blob:"],
  "font-src": ["'self'", "data:", "https://fonts.gstatic.com"],
  "connect-src": [
    "'self'",
    "https://api.example.com",
    "wss://realtime.example.com",
  ],
  "frame-src": ["'none'"],
  "object-src": ["'none'"],
  "base-uri": ["'self'"],
  "form-action": ["'self'"],
  "frame-ancestors": ["'none'"],
  "upgrade-insecure-requests": true,
};

function generateCSP(directives: CSPDirectives): string {
  return Object.entries(directives)
    .map(([key, value]) => {
      if (key === "upgrade-insecure-requests") {
        return value ? key : "";
      }
      return `${key} ${Array.isArray(value) ? value.join(" ") : value}`;
    })
    .filter(Boolean)
    .join("; ");
}

// Express middleware
function cspMiddleware(req: Request, res: Response, next: NextFunction) {
  const csp = generateCSP(cspConfig);
  res.setHeader("Content-Security-Policy", csp);

  // Report-Only mode for testing
  // res.setHeader('Content-Security-Policy-Report-Only', csp);

  next();
}

// CSP violation reporting
app.post("/csp-violation-report", (req, res) => {
  const violation = req.body;
  console.error("CSP Violation:", violation);

  // Log to monitoring service
  // monitoringService.logSecurityEvent('csp_violation', violation);

  res.status(204).send();
});

// Add report-uri to CSP
const cspWithReporting = {
  ...cspConfig,
  "report-uri": ["/csp-violation-report"],
};
```

## Security Headers

### ✅ Essential Security Headers

```typescript
// security-headers.ts
interface SecurityHeaders {
  "Strict-Transport-Security": string;
  "X-Frame-Options": string;
  "X-Content-Type-Options": string;
  "X-XSS-Protection": string;
  "Referrer-Policy": string;
  "Permissions-Policy": string;
  "Content-Security-Policy": string;
}

const securityHeaders: SecurityHeaders = {
  // HSTS: Force HTTPS for 1 year
  "Strict-Transport-Security": "max-age=31536000; includeSubDomains; preload",

  // Clickjacking protection
  "X-Frame-Options": "DENY",

  // Prevent MIME sniffing
  "X-Content-Type-Options": "nosniff",

  // XSS filter (legacy browsers)
  "X-XSS-Protection": "1; mode=block",

  // Control referrer information
  "Referrer-Policy": "strict-origin-when-cross-origin",

  // Feature policy (now Permissions Policy)
  "Permissions-Policy": [
    "geolocation=()",
    "microphone=()",
    "camera=()",
    "payment=()",
    "usb=()",
    "magnetometer=()",
    "gyroscope=()",
    "accelerometer=()",
  ].join(", "),

  // Content Security Policy
  "Content-Security-Policy": generateCSP(cspConfig),
};

// Express middleware
function securityHeadersMiddleware(
  req: Request,
  res: Response,
  next: NextFunction,
) {
  Object.entries(securityHeaders).forEach(([header, value]) => {
    res.setHeader(header, value);
  });
  next();
}

// Vite plugin for development
function viteSecurityHeaders(): Plugin {
  return {
    name: "security-headers",
    configureServer(server) {
      server.middlewares.use((req, res, next) => {
        Object.entries(securityHeaders).forEach(([header, value]) => {
          res.setHeader(header, value);
        });
        next();
      });
    },
  };
}
```

## Secrets Management

### ✅ Secure Secrets Handling

```typescript
// secrets-manager.ts

// Never commit secrets to version control!
// Use environment variables and secret management services

class SecretsManager {
  private secrets = new Map<string, string>();

  async initialize(): Promise<void> {
    // Load from environment variables
    this.loadFromEnv();

    // Or load from secret management service (AWS Secrets Manager, etc.)
    if (process.env.USE_SECRETS_MANAGER === "true") {
      await this.loadFromSecretsManager();
    }
  }

  get(key: string): string {
    const value = this.secrets.get(key);
    if (!value) {
      throw new Error(`Secret ${key} not found`);
    }
    return value;
  }

  private loadFromEnv(): void {
    const requiredSecrets = [
      "JWT_SECRET",
      "JWT_REFRESH_SECRET",
      "ENCRYPTION_KEY",
      "DATABASE_URL",
      "API_KEY",
    ];

    requiredSecrets.forEach((key) => {
      const value = process.env[key];
      if (!value) {
        throw new Error(`Required secret ${key} is not set`);
      }
      this.secrets.set(key, value);
    });
  }

  private async loadFromSecretsManager(): Promise<void> {
    // Example: AWS Secrets Manager
    // const client = new SecretsManagerClient({ region: 'us-east-1' });
    // const response = await client.send(
    //   new GetSecretValueCommand({ SecretId: 'app/secrets' })
    // );
    // const secrets = JSON.parse(response.SecretString);
    // Object.entries(secrets).forEach(([key, value]) => {
    //   this.secrets.set(key, value as string);
    // });
  }
}

// Environment variable validation
function validateEnvironment(): void {
  const required = ["NODE_ENV", "JWT_SECRET", "DATABASE_URL"];

  const missing = required.filter((key) => !process.env[key]);

  if (missing.length > 0) {
    throw new Error(
      `Missing required environment variables: ${missing.join(", ")}`,
    );
  }

  // Validate secret strength
  if (process.env.JWT_SECRET!.length < 32) {
    throw new Error("JWT_SECRET must be at least 32 characters");
  }
}

// Client-side: Never expose secrets
// Bad
// const API_KEY = 'sk_live_123456789';

// Good
// const API_KEY = process.env.VITE_PUBLIC_API_KEY; // Public key only
// Private keys must stay on server
```

## Dependency Security

### ✅ Dependency Security Practices

```json
// package.json - Security scripts
{
  "scripts": {
    "audit": "npm audit",
    "audit:fix": "npm audit fix",
    "audit:production": "npm audit --production",
    "check-updates": "npx npm-check-updates",
    "update-deps": "npx npm-check-updates -u && npm install"
  }
}
```

```yaml
# .github/workflows/security.yml
name: Security Audit

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 0 * * 0" # Weekly

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "18"

      - name: Install dependencies
        run: npm ci

      - name: Run npm audit
        run: npm audit --audit-level=moderate

      - name: Check for known vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

### ✅ Dependency Pinning

```json
// package.json - Use exact versions
{
  "dependencies": {
    "react": "18.2.0", // Exact version
    "react-dom": "18.2.0"
  },
  "devDependencies": {
    "typescript": "5.2.2"
  }
}
```

```bash
# .npmrc - Configure npm for security
save-exact=true  # Save exact versions
audit=true       # Run audit on install
fund=false       # Disable funding messages
```

## Monitoring and Incident Response

### ✅ Security Monitoring

```typescript
// security-monitor.ts
interface SecurityEvent {
  type: string;
  severity: "low" | "medium" | "high" | "critical";
  userId?: string;
  ip: string;
  userAgent: string;
  timestamp: Date;
  details: Record<string, any>;
}

class SecurityMonitor {
  private events: SecurityEvent[] = [];

  logEvent(event: Omit<SecurityEvent, "timestamp">): void {
    const fullEvent: SecurityEvent = {
      ...event,
      timestamp: new Date(),
    };

    this.events.push(fullEvent);

    // Alert on critical events
    if (event.severity === "critical") {
      this.alertSecurityTeam(fullEvent);
    }

    // Send to monitoring service
    this.sendToMonitoring(fullEvent);
  }

  async detectAnomalies(): Promise<void> {
    // Detect suspicious patterns
    await this.detectBruteForce();
    await this.detectUnusualActivity();
    await this.detectDataExfiltration();
  }

  private async detectBruteForce(): Promise<void> {
    // Count failed login attempts per IP
    const window = 5 * 60 * 1000; // 5 minutes
    const threshold = 5;

    const recentAttempts = this.events.filter(
      (e) =>
        e.type === "failed_login" &&
        Date.now() - e.timestamp.getTime() < window,
    );

    const attemptsByIP = new Map<string, number>();
    recentAttempts.forEach((event) => {
      attemptsByIP.set(event.ip, (attemptsByIP.get(event.ip) || 0) + 1);
    });

    attemptsByIP.forEach((count, ip) => {
      if (count >= threshold) {
        this.logEvent({
          type: "brute_force_detected",
          severity: "high",
          ip,
          userAgent: "",
          details: { attempts: count },
        });

        // Block IP
        this.blockIP(ip);
      }
    });
  }

  private async detectUnusualActivity(): Promise<void> {
    // Detect login from new location
    // Detect unusual data access patterns
    // Detect privilege escalation attempts
  }

  private async detectDataExfiltration(): Promise<void> {
    // Monitor large data exports
    // Detect unusual download patterns
  }

  private blockIP(ip: string): void {
    // Add to blocklist
    console.log(`Blocked IP: ${ip}`);
  }

  private alertSecurityTeam(event: SecurityEvent): void {
    // Send alert via email, Slack, PagerDuty, etc.
    console.error("SECURITY ALERT:", event);
  }

  private sendToMonitoring(event: SecurityEvent): void {
    // Send to Datadog, Sentry, etc.
  }
}

// Usage in authentication
const securityMonitor = new SecurityMonitor();

async function login(
  email: string,
  password: string,
  req: Request,
): Promise<void> {
  const user = await findUserByEmail(email);

  if (!user || !(await verifyPassword(password, user.passwordHash))) {
    securityMonitor.logEvent({
      type: "failed_login",
      severity: "medium",
      userId: user?.id,
      ip: req.ip,
      userAgent: req.headers["user-agent"] || "",
      details: { email },
    });

    throw new Error("Invalid credentials");
  }

  securityMonitor.logEvent({
    type: "successful_login",
    severity: "low",
    userId: user.id,
    ip: req.ip,
    userAgent: req.headers["user-agent"] || "",
    details: {},
  });
}
```

## Key Takeaways

1. **Defense in Depth**: Multiple layers of security controls - never rely on a single mechanism

2. **Authentication Best Practices**: Strong password policies, MFA, secure session management, JWT with short expiry

3. **Input Validation**: Validate and sanitize all user input - never trust client-side data

4. **XSS Prevention**: Escape output, use textContent over innerHTML, sanitize HTML with DOMPurify

5. **CSRF Protection**: Use CSRF tokens, SameSite cookies, verify Origin/Referer headers

6. **Encryption**: Encrypt sensitive data at rest and in transit, use strong algorithms (AES-256-GCM)

7. **Security Headers**: Implement CSP, HSTS, X-Frame-Options, and other protective headers

8. **Dependency Security**: Regular audits, update dependencies, use exact versions, monitor vulnerabilities

9. **Secrets Management**: Never commit secrets, use environment variables, rotate regularly

10. **Monitoring**: Log security events, detect anomalies, alert on suspicious activity, have incident response plan

---

**Pre-Launch Security Checklist**:

```bash
✅ Authentication
□ Password hashing (bcrypt/argon2)
□ MFA enabled
□ Session management secure
□ Token expiry configured
□ Account lockout on failures

✅ Authorization
□ RBAC/ABAC implemented
□ Least privilege principle
□ Resource-level permissions
□ API rate limiting

✅ Data Protection
□ Encryption at rest
□ HTTPS everywhere
□ Secure cookie flags
□ No sensitive data in logs/URLs

✅ Input Validation
□ Server-side validation
□ Sanitize HTML output
□ File upload restrictions
□ SQL injection prevention

✅ Security Headers
□ CSP configured
□ HSTS enabled
□ X-Frame-Options set
□ Referrer-Policy configured

✅ Monitoring
□ Security logging enabled
□ Anomaly detection active
□ Incident response plan
□ Vulnerability scanning

✅ Dependencies
□ No known vulnerabilities
□ Regular update schedule
□ License compliance

# Secure and ready! 🔒
```
