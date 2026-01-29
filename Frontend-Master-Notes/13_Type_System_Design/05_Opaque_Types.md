# 🔒 Opaque Types - Information Hiding in TypeScript

> **Hide Implementation Details and Enforce Encapsulation**

---

## 📋 Table of Contents

- [What are Opaque Types?](#what-are-opaque-types)
- [Opaque vs Branded Types](#opaque-vs-branded-types)
- [Implementation Patterns](#implementation-patterns)
- [Use Cases](#use-cases)
- [Security Applications](#security-applications)
- [Domain Modeling](#domain-modeling)
- [API Design](#api-design)
- [Testing](#testing)
- [Best Practices](#best-practices)

---

## What are Opaque Types?

Opaque types hide their internal representation, exposing only a public interface. Users can't inspect or construct values directly.

### **Transparent vs Opaque**

```typescript
// ❌ Transparent: Internal structure visible
type TransparentUserId = {
  value: string;
  createdAt: Date;
};

const id: TransparentUserId = {
  value: "user_123",
  createdAt: new Date(),
};
console.log(id.value); // Can access internals

// ✅ Opaque: Internal structure hidden
declare const UserId: unique symbol;
type OpaqueUserId = string & { [UserId]: never };

// Can't create directly
const opaqueId: OpaqueUserId = "user_123"; // ❌ Error
// Must use constructor
const validId = createUserId("user_123"); // ✅ Works
```

---

## Opaque vs Branded Types

### **Branded Types**

```typescript
// Branded: Prevents mixing, but value is accessible
type UserId = string & { __brand: "UserId" };

const userId: UserId = "user_123" as UserId;
const rawValue: string = userId; // ✅ Can access underlying string
console.log(userId.toUpperCase()); // ✅ Can use string methods
```

### **Opaque Types**

```typescript
// Opaque: Value is completely hidden
class UserId {
  private readonly __opaque!: unique symbol;

  private constructor(private readonly value: string) {}

  static create(id: string): UserId {
    return new UserId(id);
  }

  // Controlled access
  toString(): string {
    return this.value;
  }

  // No direct access to value
}

const userId = UserId.create("user_123");
const rawValue = userId.value; // ❌ Error: private property
const str = userId.toString(); // ✅ Works through public API
```

### **Comparison**

| Feature           | Branded                | Opaque            |
| ----------------- | ---------------------- | ----------------- |
| Type safety       | ✅ Yes                 | ✅ Yes            |
| Runtime overhead  | ❌ None                | ✅ Class instance |
| Value hiding      | ❌ No                  | ✅ Yes            |
| Method attachment | ❌ Prototype pollution | ✅ Class methods  |
| Serialization     | ✅ Automatic           | ⚠️ Manual         |

---

## Implementation Patterns

### **1. Class-Based Opaque Type**

```typescript
class Email {
  private readonly __opaque = "Email";

  private constructor(private readonly value: string) {}

  static create(email: string): Email {
    if (!email.includes("@")) {
      throw new Error("Invalid email");
    }
    return new Email(email);
  }

  static createUnsafe(email: string): Email {
    return new Email(email);
  }

  getDomain(): string {
    return this.value.split("@")[1];
  }

  getLocalPart(): string {
    return this.value.split("@")[0];
  }

  toString(): string {
    return this.value;
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }
}

// Usage
const email = Email.create("user@example.com");
console.log(email.getDomain()); // "example.com"
console.log(email.toString()); // "user@example.com"

const invalid = Email.create("not-an-email"); // ❌ Throws
```

### **2. Closure-Based Opaque Type**

```typescript
type UserId = ReturnType<typeof createUserId>;

function createUserId(id: string) {
  // Validate
  if (!id.startsWith("user_")) {
    throw new Error("Invalid UserId");
  }

  // Return opaque object
  return {
    toString: () => id,
    equals: (other: UserId) => other.toString() === id,
    // value is not exposed
  };
}

// Usage
const userId = createUserId("user_123");
console.log(userId.toString()); // "user_123"

// Can't access raw value
const raw = userId.value; // ❌ Error: Property 'value' doesn't exist
```

### **3. Symbol-Based Opaque Type**

```typescript
const EMAIL_SYMBOL = Symbol("Email");

interface Email {
  [EMAIL_SYMBOL]: string;
}

function createEmail(email: string): Email {
  if (!email.includes("@")) {
    throw new Error("Invalid email");
  }

  return {
    [EMAIL_SYMBOL]: email,
  };
}

function getEmailValue(email: Email): string {
  return email[EMAIL_SYMBOL];
}

// Usage
const email = createEmail("user@example.com");
const value = getEmailValue(email); // ✅ Controlled access

// Can't access directly
const raw = email[EMAIL_SYMBOL]; // ❌ Error in strict mode
```

### **4. Branded with Private Access**

```typescript
const UserIdBrand = Symbol("UserId");

class UserIdImpl {
  private constructor(private value: string) {}

  static [UserIdBrand](value: string): UserId {
    return new UserIdImpl(value) as unknown as UserId;
  }

  toString(this: UserId): string {
    return (this as unknown as UserIdImpl).value;
  }
}

// Opaque type - can't see UserIdImpl
type UserId = { readonly [UserIdBrand]: unique symbol };

// Public API
const createUserId = (id: string) => UserIdImpl[UserIdBrand](id);

const userIdToString = (id: UserId) => UserIdImpl.prototype.toString.call(id);

// Usage
const userId = createUserId("user_123");
const str = userIdToString(userId); // ✅ Works
const raw = userId.value; // ❌ Error: no 'value' property
```

---

## Use Cases

### **1. Validated Strings**

```typescript
class PositiveInteger {
  private readonly __opaque = "PositiveInteger";

  private constructor(private readonly value: number) {}

  static create(n: number): PositiveInteger {
    if (!Number.isInteger(n) || n <= 0) {
      throw new Error("Must be positive integer");
    }
    return new PositiveInteger(n);
  }

  add(other: PositiveInteger): PositiveInteger {
    return new PositiveInteger(this.value + other.value);
  }

  multiply(other: PositiveInteger): PositiveInteger {
    return new PositiveInteger(this.value * other.value);
  }

  toNumber(): number {
    return this.value;
  }
}

// Usage - can only create valid values
const a = PositiveInteger.create(5);
const b = PositiveInteger.create(3);
const sum = a.add(b); // PositiveInteger(8)

const invalid = PositiveInteger.create(-1); // ❌ Throws
const alsoInvalid = PositiveInteger.create(3.14); // ❌ Throws
```

### **2. Units of Measurement**

```typescript
class Meters {
  private readonly __opaque = "Meters";

  private constructor(private readonly value: number) {}

  static create(meters: number): Meters {
    if (meters < 0) throw new Error("Negative distance");
    return new Meters(meters);
  }

  static fromKilometers(km: number): Meters {
    return new Meters(km * 1000);
  }

  static fromMiles(miles: number): Meters {
    return new Meters(miles * 1609.34);
  }

  toKilometers(): number {
    return this.value / 1000;
  }

  toMiles(): number {
    return this.value / 1609.34;
  }

  add(other: Meters): Meters {
    return new Meters(this.value + other.value);
  }
}

// Usage - prevents unit confusion
const distance1 = Meters.create(1000);
const distance2 = Meters.fromKilometers(2);
const total = distance1.add(distance2); // Type-safe addition

// Can't mix with plain numbers
const wrong = distance1 + 500; // ❌ Type error
```

### **3. Timestamps**

```typescript
class Timestamp {
  private readonly __opaque = "Timestamp";

  private constructor(private readonly ms: number) {}

  static now(): Timestamp {
    return new Timestamp(Date.now());
  }

  static fromDate(date: Date): Timestamp {
    return new Timestamp(date.getTime());
  }

  static fromMilliseconds(ms: number): Timestamp {
    return new Timestamp(ms);
  }

  isBefore(other: Timestamp): boolean {
    return this.ms < other.ms;
  }

  isAfter(other: Timestamp): boolean {
    return this.ms > other.ms;
  }

  differenceMs(other: Timestamp): number {
    return Math.abs(this.ms - other.ms);
  }

  toDate(): Date {
    return new Date(this.ms);
  }

  toISOString(): string {
    return new Date(this.ms).toISOString();
  }
}

// Usage
const created = Timestamp.now();
const updated = Timestamp.fromDate(new Date());

if (updated.isAfter(created)) {
  console.log("Updated after creation");
}
```

---

## Security Applications

### **1. Sanitized HTML**

```typescript
class SafeHtml {
  private readonly __opaque = "SafeHtml";

  private constructor(private readonly html: string) {}

  static sanitize(html: string): SafeHtml {
    // Use DOMPurify or similar
    const clean = DOMPurify.sanitize(html, {
      ALLOWED_TAGS: ["b", "i", "em", "strong", "a"],
      ALLOWED_ATTR: ["href"],
    });
    return new SafeHtml(clean);
  }

  static trusted(html: string): SafeHtml {
    // For template literals from trusted sources
    return new SafeHtml(html);
  }

  toString(): string {
    return this.html;
  }

  concat(other: SafeHtml): SafeHtml {
    return new SafeHtml(this.html + other.html);
  }
}

// Usage
function render(content: SafeHtml) {
  document.body.innerHTML = content.toString(); // Safe!
}

// Must sanitize user input
const userInput = "<script>alert('xss')</script>";
const safe = SafeHtml.sanitize(userInput);
render(safe); // ✅ XSS prevented

// Can't pass raw strings
render("<div>Hello</div>"); // ❌ Type error
render(SafeHtml.trusted("<div>Hello</div>")); // ✅ Explicit opt-in
```

### **2. Encrypted Data**

```typescript
class EncryptedString {
  private readonly __opaque = "EncryptedString";

  private constructor(private readonly ciphertext: string) {}

  static async encrypt(
    plaintext: string,
    key: CryptoKey,
  ): Promise<EncryptedString> {
    const encrypted = await crypto.subtle.encrypt(
      { name: "AES-GCM", iv: generateIV() },
      key,
      new TextEncoder().encode(plaintext),
    );

    const ciphertext = btoa(String.fromCharCode(...new Uint8Array(encrypted)));
    return new EncryptedString(ciphertext);
  }

  async decrypt(key: CryptoKey): Promise<string> {
    const data = Uint8Array.from(atob(this.ciphertext), (c) => c.charCodeAt(0));

    const decrypted = await crypto.subtle.decrypt(
      { name: "AES-GCM", iv: extractIV(this.ciphertext) },
      key,
      data,
    );

    return new TextDecoder().decode(decrypted);
  }

  // No way to get plaintext without key
  getCiphertext(): string {
    return this.ciphertext;
  }
}

// Usage
const key = await generateKey();
const secret = "sensitive data";

// Must encrypt before storage
const encrypted = await EncryptedString.encrypt(secret, key);
await database.save(encrypted.getCiphertext());

// Must decrypt to use
const retrieved = new EncryptedString(await database.load());
const plaintext = await retrieved.decrypt(key);
```

### **3. Access Tokens**

```typescript
class AccessToken {
  private readonly __opaque = "AccessToken";

  private constructor(
    private readonly token: string,
    private readonly expiresAt: Date,
  ) {}

  static create(token: string, expiresIn: number): AccessToken {
    const expiresAt = new Date(Date.now() + expiresIn * 1000);
    return new AccessToken(token, expiresAt);
  }

  isExpired(): boolean {
    return new Date() > this.expiresAt;
  }

  isValid(): boolean {
    return !this.isExpired();
  }

  getHeader(): string {
    if (this.isExpired()) {
      throw new Error("Token expired");
    }
    return `Bearer ${this.token}`;
  }

  // No direct access to token
}

// Usage
function makeAuthenticatedRequest(
  url: string,
  token: AccessToken,
): Promise<Response> {
  if (!token.isValid()) {
    throw new Error("Invalid token");
  }

  return fetch(url, {
    headers: {
      Authorization: token.getHeader(),
    },
  });
}

// Can't pass raw strings
const response = await makeAuthenticatedRequest(
  "/api/data",
  "raw-token-string", // ❌ Type error
);
```

---

## Domain Modeling

### **1. Money**

```typescript
class Money {
  private readonly __opaque = "Money";

  private constructor(
    private readonly amount: number,
    private readonly currency: string,
  ) {}

  static create(amount: number, currency: string): Money {
    if (amount < 0) {
      throw new Error("Amount cannot be negative");
    }
    if (!["USD", "EUR", "GBP"].includes(currency)) {
      throw new Error("Unsupported currency");
    }
    return new Money(amount, currency);
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error("Currency mismatch");
    }
    return new Money(this.amount + other.amount, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(this.amount * factor, this.currency);
  }

  format(): string {
    return new Intl.NumberFormat("en-US", {
      style: "currency",
      currency: this.currency,
    }).format(this.amount);
  }

  getCurrency(): string {
    return this.currency;
  }
}

// Usage - prevents currency mistakes
const price = Money.create(99.99, "USD");
const tax = price.multiply(0.08);
const total = price.add(tax); // Type-safe

console.log(total.format()); // "$107.99"

// Can't add different currencies
const euros = Money.create(50, "EUR");
price.add(euros); // ❌ Throws: Currency mismatch
```

### **2. Email Address**

```typescript
class EmailAddress {
  private readonly __opaque = "EmailAddress";

  private constructor(private readonly email: string) {}

  static create(email: string): EmailAddress {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(email)) {
      throw new Error("Invalid email address");
    }
    return new EmailAddress(email.toLowerCase());
  }

  getDomain(): string {
    return this.email.split("@")[1];
  }

  getLocalPart(): string {
    return this.email.split("@")[0];
  }

  toString(): string {
    return this.email;
  }

  equals(other: EmailAddress): boolean {
    return this.email === other.email;
  }

  // Check if disposable email
  isDisposable(): boolean {
    const disposableDomains = ["tempmail.com", "10minutemail.com"];
    return disposableDomains.includes(this.getDomain());
  }
}

// Usage
function sendEmail(to: EmailAddress, subject: string, body: string) {
  if (to.isDisposable()) {
    throw new Error("Disposable emails not allowed");
  }

  // Send email to to.toString()
}

const email = EmailAddress.create("user@example.com");
sendEmail(email, "Hello", "World");

// Can't pass invalid emails
sendEmail("invalid-email", "Hello", "World"); // ❌ Type error
```

---

## API Design

### **1. Pagination Token**

```typescript
class PaginationToken {
  private readonly __opaque = "PaginationToken";

  private constructor(private readonly encoded: string) {}

  static encode(data: { offset: number; limit: number }): PaginationToken {
    const json = JSON.stringify(data);
    const base64 = btoa(json);
    return new PaginationToken(base64);
  }

  decode(): { offset: number; limit: number } {
    const json = atob(this.encoded);
    return JSON.parse(json);
  }

  toString(): string {
    return this.encoded;
  }

  static fromString(token: string): PaginationToken {
    try {
      // Validate it's valid base64
      atob(token);
      return new PaginationToken(token);
    } catch {
      throw new Error("Invalid pagination token");
    }
  }
}

// API usage
interface PaginatedResponse<T> {
  items: T[];
  nextToken?: PaginationToken;
}

async function getUsers(
  token?: PaginationToken,
): Promise<PaginatedResponse<User>> {
  const { offset, limit } = token?.decode() ?? { offset: 0, limit: 10 };

  const users = await db.users.findMany({
    skip: offset,
    take: limit,
  });

  const nextToken =
    users.length === limit
      ? PaginationToken.encode({ offset: offset + limit, limit })
      : undefined;

  return { items: users, nextToken };
}

// Client usage
let token: PaginationToken | undefined;
do {
  const response = await getUsers(token);
  console.log(response.items);
  token = response.nextToken;
} while (token);
```

### **2. API Key**

```typescript
class ApiKey {
  private readonly __opaque = "ApiKey";
  private static readonly PREFIX = "sk_live_";

  private constructor(private readonly key: string) {}

  static generate(): ApiKey {
    const randomBytes = crypto.getRandomValues(new Uint8Array(32));
    const key = ApiKey.PREFIX + btoa(String.fromCharCode(...randomBytes));
    return new ApiKey(key);
  }

  static fromString(key: string): ApiKey {
    if (!key.startsWith(ApiKey.PREFIX)) {
      throw new Error("Invalid API key format");
    }
    return new ApiKey(key);
  }

  toString(): string {
    return this.key;
  }

  // Redact for logging
  toLogString(): string {
    return `${ApiKey.PREFIX}****${this.key.slice(-4)}`;
  }

  equals(other: ApiKey): boolean {
    return this.key === other.key;
  }
}

// Usage
const apiKey = ApiKey.generate();
console.log(apiKey.toLogString()); // "sk_live_****xyz"

// Store securely
await saveToDatabase(apiKey.toString());

// Load and validate
const loaded = ApiKey.fromString(await loadFromDatabase());
```

---

## Testing

### **1. Test Utilities**

```typescript
class Email {
  private readonly __opaque = "Email";
  private constructor(private readonly value: string) {}

  static create(email: string): Email {
    if (!email.includes("@")) throw new Error("Invalid");
    return new Email(email);
  }

  toString(): string {
    return this.value;
  }
}

// Test utilities
export const TestEmail = {
  // Create without validation for tests
  unsafe: (email: string) => Email.create(email),

  // Common test emails
  valid: () => Email.create("test@example.com"),
  invalid: () => "not-an-email", // Returns string, will cause type error

  // Builder for tests
  build: (local: string = "test", domain: string = "example.com") =>
    Email.create(`${local}@${domain}`),
};

// Usage in tests
describe("Email", () => {
  it("should create valid email", () => {
    const email = TestEmail.valid();
    expect(email.toString()).toBe("test@example.com");
  });

  it("should reject invalid email", () => {
    expect(() => Email.create("invalid")).toThrow();
  });

  it("should use custom email", () => {
    const email = TestEmail.build("john", "doe.com");
    expect(email.toString()).toBe("john@doe.com");
  });
});
```

### **2. Mocking**

```typescript
class UserId {
  private readonly __opaque = "UserId";
  private constructor(private readonly id: string) {}

  static create(id: string): UserId {
    return new UserId(id);
  }

  toString(): string {
    return this.id;
  }
}

// Mock for tests
export class MockUserId {
  static create(id: string = "mock-user-id"): UserId {
    return UserId.create(id);
  }

  static sequence(start: number = 1) {
    let counter = start;
    return () => UserId.create(`user-${counter++}`);
  }
}

// Usage
describe("User service", () => {
  it("should handle multiple users", () => {
    const nextId = MockUserId.sequence();

    const user1 = createUser(nextId()); // user-1
    const user2 = createUser(nextId()); // user-2
    const user3 = createUser(nextId()); // user-3
  });
});
```

---

## Best Practices

### **1. Provide factory functions**

```typescript
// ✅ Good: Factory method
class Email {
  private constructor(private value: string) {}

  static create(email: string): Email {
    // Validation
    return new Email(email);
  }
}

// ❌ Bad: Public constructor
class BadEmail {
  constructor(public value: string) {
    // Can't enforce validation
  }
}
```

### **2. Make constructors private**

```typescript
// ✅ Good: Can't bypass validation
class UserId {
  private constructor(private id: string) {}

  static create(id: string): UserId {
    if (!id) throw new Error("Empty ID");
    return new UserId(id);
  }
}

new UserId("anything"); // ❌ Error: private constructor
```

### **3. Provide controlled access**

```typescript
// ✅ Good: Expose what's needed
class Money {
  private constructor(
    private amount: number,
    private currency: string,
  ) {}

  // Expose operations, not data
  add(other: Money): Money {
    /* ... */
  }
  format(): string {
    /* ... */
  }
  getCurrency(): string {
    return this.currency;
  }

  // Don't expose raw amount
  // getAmount(): number { return this.amount; }
}
```

### **4. Use readonly fields**

```typescript
class Email {
  // ✅ Good: Can't be modified
  private constructor(private readonly value: string) {}
}
```

### **5. Consider serialization**

```typescript
class UserId {
  private constructor(private id: string) {}

  static create(id: string): UserId {
    return new UserId(id);
  }

  // ✅ Support JSON serialization
  toJSON(): string {
    return this.id;
  }

  static fromJSON(json: string): UserId {
    return new UserId(json);
  }
}

// Usage
const userId = UserId.create("user_123");
const json = JSON.stringify(userId); // "user_123"
const restored = UserId.fromJSON(JSON.parse(json));
```

---

## Summary

### **Key Benefits**

- ✅ Complete encapsulation
- ✅ Enforced validation
- ✅ Type-safe operations
- ✅ Clear API boundaries
- ✅ Security by design

### **When to Use**

- Validated data (emails, URLs, IDs)
- Security-sensitive values (tokens, passwords)
- Domain concepts (Money, measurements)
- Encoded data (pagination tokens, API keys)

### **Trade-offs**

**Pros:**

- Strong encapsulation
- Enforced invariants
- Clear interface
- Better security

**Cons:**

- Runtime overhead (class instances)
- More boilerplate
- Serialization complexity
- Learning curve

---

**Interview Questions:**

1. What's the difference between branded and opaque types?
2. How do opaque types improve security?
3. What's the runtime cost of opaque types?
4. How do you serialize opaque types?

**Practice:**

- Convert branded types to opaque
- Implement Money type
- Create secure token type
- Build validated email class
