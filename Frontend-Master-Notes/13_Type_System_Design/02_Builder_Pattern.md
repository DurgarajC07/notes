# 🏗️ Builder Pattern with TypeScript

> **Fluent APIs with Type Safety - Building Complex Objects Step by Step**

---

## 📋 Table of Contents

- [What is Builder Pattern?](#what-is-builder-pattern)
- [Why Use Builder Pattern?](#why-use-builder-pattern)
- [Basic Implementation](#basic-implementation)
- [Advanced Patterns](#advanced-patterns)
- [Type-Safe Builders](#type-safe-builders)
- [Required vs Optional Fields](#required-vs-optional-fields)
- [Method Chaining](#method-chaining)
- [Real-World Examples](#real-world-examples)
- [Common Pitfalls](#common-pitfalls)
- [Best Practices](#best-practices)

---

## What is Builder Pattern?

The Builder pattern separates construction of complex objects from their representation, allowing step-by-step creation with fluent APIs.

### **Key Concepts**

```typescript
// Complex object we want to build
interface User {
  id: string;
  name: string;
  email: string;
  age?: number;
  address?: Address;
  preferences?: UserPreferences;
}

// Builder provides fluent API
class UserBuilder {
  private user: Partial<User> = {};

  setId(id: string): this {
    this.user.id = id;
    return this;
  }

  setName(name: string): this {
    this.user.name = name;
    return this;
  }

  setEmail(email: string): this {
    this.user.email = email;
    return this;
  }

  build(): User {
    if (!this.user.id || !this.user.name || !this.user.email) {
      throw new Error("Missing required fields");
    }
    return this.user as User;
  }
}

// Usage
const user = new UserBuilder()
  .setId("123")
  .setName("John")
  .setEmail("john@example.com")
  .build();
```

---

## Why Use Builder Pattern?

### **1. Readability**

```typescript
// ❌ Constructor with many parameters
const user = new User(
  "123",
  "John",
  "john@example.com",
  25,
  undefined,
  undefined,
  { theme: "dark" },
);

// ✅ Builder pattern
const user = new UserBuilder()
  .setId("123")
  .setName("John")
  .setEmail("john@example.com")
  .setAge(25)
  .setPreferences({ theme: "dark" })
  .build();
```

### **2. Optional Parameters**

```typescript
// No need to pass undefined for optional fields
const simpleUser = new UserBuilder()
  .setId("456")
  .setName("Jane")
  .setEmail("jane@example.com")
  .build();
```

### **3. Validation**

```typescript
class ValidatedUserBuilder {
  private user: Partial<User> = {};

  setEmail(email: string): this {
    if (!email.includes("@")) {
      throw new Error("Invalid email");
    }
    this.user.email = email;
    return this;
  }

  setAge(age: number): this {
    if (age < 0 || age > 150) {
      throw new Error("Invalid age");
    }
    this.user.age = age;
    return this;
  }

  build(): User {
    // Final validation
    if (!this.user.id || !this.user.name || !this.user.email) {
      throw new Error("Missing required fields");
    }
    return this.user as User;
  }
}
```

---

## Basic Implementation

### **Generic Builder**

```typescript
class Builder<T> {
  private instance: Partial<T> = {};

  set<K extends keyof T>(key: K, value: T[K]): this {
    this.instance[key] = value;
    return this;
  }

  build(): T {
    return this.instance as T;
  }
}

// Usage
interface Product {
  id: number;
  name: string;
  price: number;
}

const product = new Builder<Product>()
  .set("id", 1)
  .set("name", "Laptop")
  .set("price", 999)
  .build();
```

### **Method-Based Builder**

```typescript
class QueryBuilder {
  private query: {
    select?: string[];
    from?: string;
    where?: string[];
    orderBy?: string;
    limit?: number;
  } = {};

  select(...fields: string[]): this {
    this.query.select = fields;
    return this;
  }

  from(table: string): this {
    this.query.from = table;
    return this;
  }

  where(condition: string): this {
    if (!this.query.where) {
      this.query.where = [];
    }
    this.query.where.push(condition);
    return this;
  }

  orderBy(field: string): this {
    this.query.orderBy = field;
    return this;
  }

  limit(count: number): this {
    this.query.limit = count;
    return this;
  }

  build(): string {
    const { select = ["*"], from, where = [], orderBy, limit } = this.query;

    if (!from) {
      throw new Error("FROM clause is required");
    }

    let sql = `SELECT ${select.join(", ")} FROM ${from}`;

    if (where.length > 0) {
      sql += ` WHERE ${where.join(" AND ")}`;
    }

    if (orderBy) {
      sql += ` ORDER BY ${orderBy}`;
    }

    if (limit) {
      sql += ` LIMIT ${limit}`;
    }

    return sql;
  }
}

// Usage
const sql = new QueryBuilder()
  .select("name", "email")
  .from("users")
  .where("age > 18")
  .where("active = true")
  .orderBy("name")
  .limit(10)
  .build();

console.log(sql);
// SELECT name, email FROM users WHERE age > 18 AND active = true ORDER BY name LIMIT 10
```

---

## Advanced Patterns

### **Type-State Builder Pattern**

Enforces required fields at compile time:

```typescript
// Define states
interface NoId {
  _tag: "NoId";
}

interface HasId {
  _tag: "HasId";
  id: string;
}

interface NoName {
  _tag: "NoName";
}

interface HasName {
  _tag: "HasName";
  name: string;
}

// Builder tracks state with phantom types
class TypeSafeUserBuilder<
  IdState extends NoId | HasId = NoId,
  NameState extends NoName | HasName = NoName,
> {
  private data: Partial<User> = {};

  setId(id: string): TypeSafeUserBuilder<HasId, NameState> {
    this.data.id = id;
    return this as any;
  }

  setName(name: string): TypeSafeUserBuilder<IdState, HasName> {
    this.data.name = name;
    return this as any;
  }

  setEmail(email: string): this {
    this.data.email = email;
    return this;
  }

  // Can only build when all required fields are set
  build(this: TypeSafeUserBuilder<HasId, HasName>): User {
    return this.data as User;
  }
}

// ✅ Compiles - all required fields set
const user1 = new TypeSafeUserBuilder()
  .setId("123")
  .setName("John")
  .setEmail("john@example.com")
  .build();

// ❌ Compile error - missing name
const user2 = new TypeSafeUserBuilder()
  .setId("123")
  .setEmail("john@example.com")
  .build(); // Error: Property 'build' does not exist
```

### **Simplified Type-State Pattern**

```typescript
type BuilderState<T, Required extends keyof T> = {
  [K in keyof T]?: T[K];
} & {
  _required: Required;
};

class SmartBuilder<T, Required extends keyof T = never> {
  private data: Partial<T> = {};

  set<K extends keyof T>(key: K, value: T[K]): SmartBuilder<T, Required | K> {
    this.data[key] = value;
    return this as any;
  }

  build(this: SmartBuilder<T, keyof T>): T {
    return this.data as T;
  }
}

interface Config {
  host: string;
  port: number;
  ssl: boolean;
}

// ✅ All required fields set
const config1 = new SmartBuilder<Config>()
  .set("host", "localhost")
  .set("port", 3000)
  .set("ssl", true)
  .build();

// ❌ Missing 'ssl'
const config2 = new SmartBuilder<Config>()
  .set("host", "localhost")
  .set("port", 3000)
  .build(); // Error
```

---

## Type-Safe Builders

### **With Branded Types**

```typescript
type Brand<T, B> = T & { __brand: B };
type UserId = Brand<string, "UserId">;
type Email = Brand<string, "Email">;

class BrandedUserBuilder {
  private user: Partial<{
    id: UserId;
    email: Email;
    name: string;
  }> = {};

  setId(id: UserId): this {
    this.user.id = id;
    return this;
  }

  setEmail(email: Email): this {
    this.user.email = email;
    return this;
  }

  setName(name: string): this {
    this.user.name = name;
    return this;
  }

  build() {
    return this.user;
  }
}

// Type-safe factories
const createUserId = (id: string): UserId => id as UserId;
const createEmail = (email: string): Email => {
  if (!email.includes("@")) {
    throw new Error("Invalid email");
  }
  return email as Email;
};

// ✅ Type-safe usage
const user = new BrandedUserBuilder()
  .setId(createUserId("123"))
  .setEmail(createEmail("john@example.com"))
  .setName("John")
  .build();

// ❌ Can't pass raw strings
const badUser = new BrandedUserBuilder()
  .setId("123") // Error: Type 'string' is not assignable to type 'UserId'
  .build();
```

### **With Validation**

```typescript
class ValidatingBuilder<T> {
  private data: Partial<T> = {};
  private validators: Map<keyof T, (value: any) => boolean> = new Map();

  addValidator<K extends keyof T>(
    key: K,
    validator: (value: T[K]) => boolean,
  ): this {
    this.validators.set(key, validator);
    return this;
  }

  set<K extends keyof T>(key: K, value: T[K]): this {
    const validator = this.validators.get(key);
    if (validator && !validator(value)) {
      throw new Error(`Validation failed for ${String(key)}`);
    }
    this.data[key] = value;
    return this;
  }

  build(): T {
    return this.data as T;
  }
}

interface Person {
  name: string;
  age: number;
  email: string;
}

const person = new ValidatingBuilder<Person>()
  .addValidator("age", (age) => age > 0 && age < 150)
  .addValidator("email", (email) => email.includes("@"))
  .set("name", "John")
  .set("age", 30)
  .set("email", "john@example.com")
  .build();
```

---

## Required vs Optional Fields

### **Separate Required and Optional**

```typescript
interface RequiredUser {
  id: string;
  name: string;
  email: string;
}

interface OptionalUserFields {
  age?: number;
  address?: string;
  phone?: string;
}

type User = RequiredUser & OptionalUserFields;

class SeparatedBuilder {
  private required: Partial<RequiredUser> = {};
  private optional: OptionalUserFields = {};

  // Required methods return builder with tracking
  setId(id: string): this {
    this.required.id = id;
    return this;
  }

  setName(name: string): this {
    this.required.name = name;
    return this;
  }

  setEmail(email: string): this {
    this.required.email = email;
    return this;
  }

  // Optional methods
  setAge(age: number): this {
    this.optional.age = age;
    return this;
  }

  setAddress(address: string): this {
    this.optional.address = address;
    return this;
  }

  build(): User {
    // Check all required fields
    if (!this.required.id || !this.required.name || !this.required.email) {
      throw new Error("Missing required fields");
    }

    return {
      ...(this.required as RequiredUser),
      ...this.optional,
    };
  }
}
```

### **With Default Values**

```typescript
class ConfigBuilder {
  private config: {
    host: string;
    port: number;
    ssl: boolean;
    timeout: number;
    retries: number;
  };

  constructor() {
    // Initialize with defaults
    this.config = {
      host: "localhost",
      port: 3000,
      ssl: false,
      timeout: 5000,
      retries: 3,
    };
  }

  setHost(host: string): this {
    this.config.host = host;
    return this;
  }

  setPort(port: number): this {
    this.config.port = port;
    return this;
  }

  enableSsl(): this {
    this.config.ssl = true;
    return this;
  }

  setTimeout(timeout: number): this {
    this.config.timeout = timeout;
    return this;
  }

  setRetries(retries: number): this {
    this.config.retries = retries;
    return this;
  }

  build() {
    return { ...this.config };
  }
}

// Only override what you need
const config = new ConfigBuilder()
  .setHost("api.example.com")
  .enableSsl()
  .build();
// { host: 'api.example.com', port: 3000, ssl: true, timeout: 5000, retries: 3 }
```

---

## Method Chaining

### **Conditional Chaining**

```typescript
class ConditionalBuilder {
  private config: Record<string, any> = {};

  set(key: string, value: any): this {
    this.config[key] = value;
    return this;
  }

  setIf(condition: boolean, key: string, value: any): this {
    if (condition) {
      this.config[key] = value;
    }
    return this;
  }

  setWhen<T>(
    value: T | undefined | null,
    setter: (builder: this, value: T) => this,
  ): this {
    if (value != null) {
      return setter(this, value);
    }
    return this;
  }

  build() {
    return this.config;
  }
}

// Usage
const isProd = process.env.NODE_ENV === "production";
const apiKey = process.env.API_KEY;

const config = new ConditionalBuilder()
  .set("host", "api.example.com")
  .setIf(isProd, "ssl", true)
  .setIf(!isProd, "debug", true)
  .setWhen(apiKey, (builder, key) => builder.set("apiKey", key))
  .build();
```

### **Nested Builders**

```typescript
class AddressBuilder {
  private address: Partial<Address> = {};

  setStreet(street: string): this {
    this.address.street = street;
    return this;
  }

  setCity(city: string): this {
    this.address.city = city;
    return this;
  }

  setZip(zip: string): this {
    this.address.zip = zip;
    return this;
  }

  build(): Address {
    return this.address as Address;
  }
}

class UserWithAddressBuilder {
  private user: Partial<User> = {};

  setName(name: string): this {
    this.user.name = name;
    return this;
  }

  setEmail(email: string): this {
    this.user.email = email;
    return this;
  }

  withAddress(builderFn: (builder: AddressBuilder) => AddressBuilder): this {
    const addressBuilder = new AddressBuilder();
    const address = builderFn(addressBuilder).build();
    this.user.address = address;
    return this;
  }

  build(): User {
    return this.user as User;
  }
}

// Usage
const user = new UserWithAddressBuilder()
  .setName("John")
  .setEmail("john@example.com")
  .withAddress((addr) =>
    addr.setStreet("123 Main St").setCity("New York").setZip("10001"),
  )
  .build();
```

---

## Real-World Examples

### **HTTP Request Builder**

```typescript
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";

interface RequestConfig {
  url: string;
  method: HttpMethod;
  headers?: Record<string, string>;
  body?: any;
  timeout?: number;
  retries?: number;
}

class HttpRequestBuilder {
  private config: Partial<RequestConfig> = {
    method: "GET",
    headers: {},
    timeout: 30000,
    retries: 0,
  };

  url(url: string): this {
    this.config.url = url;
    return this;
  }

  get(): this {
    this.config.method = "GET";
    return this;
  }

  post(): this {
    this.config.method = "POST";
    return this;
  }

  put(): this {
    this.config.method = "PUT";
    return this;
  }

  delete(): this {
    this.config.method = "DELETE";
    return this;
  }

  header(key: string, value: string): this {
    this.config.headers![key] = value;
    return this;
  }

  headers(headers: Record<string, string>): this {
    this.config.headers = { ...this.config.headers, ...headers };
    return this;
  }

  body(body: any): this {
    this.config.body = body;
    return this;
  }

  json(data: any): this {
    this.header("Content-Type", "application/json");
    this.config.body = JSON.stringify(data);
    return this;
  }

  timeout(ms: number): this {
    this.config.timeout = ms;
    return this;
  }

  retries(count: number): this {
    this.config.retries = count;
    return this;
  }

  auth(token: string): this {
    return this.header("Authorization", `Bearer ${token}`);
  }

  async send(): Promise<Response> {
    if (!this.config.url) {
      throw new Error("URL is required");
    }

    const { url, method, headers, body, timeout, retries } = this.config;

    let lastError: Error | null = null;

    for (let attempt = 0; attempt <= retries!; attempt++) {
      try {
        const controller = new AbortController();
        const timeoutId = setTimeout(() => controller.abort(), timeout);

        const response = await fetch(url, {
          method,
          headers,
          body,
          signal: controller.signal,
        });

        clearTimeout(timeoutId);
        return response;
      } catch (error) {
        lastError = error as Error;
        if (attempt < retries!) {
          await new Promise((resolve) =>
            setTimeout(resolve, 1000 * (attempt + 1)),
          );
        }
      }
    }

    throw lastError;
  }
}

// Usage
const response = await new HttpRequestBuilder()
  .url("https://api.example.com/users")
  .post()
  .json({ name: "John", email: "john@example.com" })
  .auth("my-token")
  .timeout(5000)
  .retries(3)
  .send();

const user = await response.json();
```

### **React Component Builder**

```typescript
interface ComponentProps {
  className?: string;
  style?: React.CSSProperties;
  onClick?: () => void;
  children?: React.ReactNode;
}

class ComponentBuilder<P extends ComponentProps> {
  private props: P = {} as P;

  className(className: string): this {
    this.props.className = className as any;
    return this;
  }

  addClassName(className: string): this {
    this.props.className = `${this.props.className || ''} ${className}`.trim() as any;
    return this;
  }

  style(style: React.CSSProperties): this {
    this.props.style = { ...this.props.style, ...style } as any;
    return this;
  }

  onClick(handler: () => void): this {
    this.props.onClick = handler as any;
    return this;
  }

  children(children: React.ReactNode): this {
    this.props.children = children as any;
    return this;
  }

  build(): P {
    return this.props;
  }
}

// Usage in component
function MyComponent() {
  const buttonProps = new ComponentBuilder()
    .className('btn')
    .addClassName('btn-primary')
    .style({ padding: '10px 20px' })
    .onClick(() => console.log('Clicked'))
    .children('Click Me')
    .build();

  return <button {...buttonProps} />;
}
```

### **Test Data Builder**

```typescript
class UserTestBuilder {
  private user: Partial<User> = {
    id: "test-id-" + Math.random(),
    name: "Test User",
    email: "test@example.com",
    createdAt: new Date(),
  };

  withId(id: string): this {
    this.user.id = id;
    return this;
  }

  withName(name: string): this {
    this.user.name = name;
    return this;
  }

  withEmail(email: string): this {
    this.user.email = email;
    return this;
  }

  asAdmin(): this {
    this.user.role = "admin";
    return this;
  }

  asGuest(): this {
    this.user.role = "guest";
    return this;
  }

  withPremium(): this {
    this.user.subscription = "premium";
    return this;
  }

  build(): User {
    return this.user as User;
  }
}

// Usage in tests
describe("User tests", () => {
  it("should allow admin to delete users", () => {
    const admin = new UserTestBuilder().asAdmin().build();

    const result = deleteUser(admin, "user-123");
    expect(result).toBe(true);
  });

  it("should not allow guest to delete users", () => {
    const guest = new UserTestBuilder().asGuest().build();

    const result = deleteUser(guest, "user-123");
    expect(result).toBe(false);
  });
});
```

---

## Common Pitfalls

### **1. Mutable State**

```typescript
// ❌ Bad: Mutable builder
class BadBuilder {
  private config = { count: 0 };

  increment(): this {
    this.config.count++;
    return this;
  }

  build() {
    return this.config; // Returns same reference
  }
}

const builder = new BadBuilder();
const config1 = builder.increment().build();
const config2 = builder.increment().build();
console.log(config1 === config2); // true - same object!

// ✅ Good: Return new object
class GoodBuilder {
  private config = { count: 0 };

  increment(): this {
    this.config.count++;
    return this;
  }

  build() {
    return { ...this.config }; // New object
  }
}
```

### **2. Missing Validation**

```typescript
// ❌ Bad: No validation
class NoValidationBuilder {
  private email = "";

  setEmail(email: string): this {
    this.email = email;
    return this;
  }

  build() {
    return { email: this.email };
  }
}

// ✅ Good: Validate early
class ValidatedBuilder {
  private email = "";

  setEmail(email: string): this {
    if (!email.includes("@")) {
      throw new Error("Invalid email");
    }
    this.email = email;
    return this;
  }

  build() {
    if (!this.email) {
      throw new Error("Email is required");
    }
    return { email: this.email };
  }
}
```

### **3. Over-Engineering**

```typescript
// ❌ Bad: Builder for simple object
interface SimpleConfig {
  host: string;
  port: number;
}

class OverEngineeredBuilder {
  // ... 50 lines of builder code ...
}

// ✅ Good: Just use object literal
const config: SimpleConfig = {
  host: "localhost",
  port: 3000,
};
```

---

## Best Practices

### **1. Use for Complex Objects Only**

```typescript
// ✅ Good use case: Many optional fields
new HttpRequestBuilder()
  .url("/api/users")
  .post()
  .json(data)
  .auth(token)
  .timeout(5000)
  .retries(3)
  .send();

// ❌ Bad use case: Simple object
new SimpleBuilder().setA(1).setB(2).build();
// Just use: { a: 1, b: 2 }
```

### **2. Return `this` for Chaining**

```typescript
class ChainableBuilder {
  setName(name: string): this {
    // ...
    return this; // Always return this
  }
}
```

### **3. Validate in build()**

```typescript
class SafeBuilder {
  build(): User {
    // Perform final validation
    if (!this.isValid()) {
      throw new Error("Invalid configuration");
    }
    return this.data as User;
  }

  private isValid(): boolean {
    // Check all invariants
    return true;
  }
}
```

### **4. Consider Immutability**

```typescript
class ImmutableBuilder {
  constructor(private readonly data: Readonly<Partial<Config>>) {}

  setHost(host: string): ImmutableBuilder {
    return new ImmutableBuilder({ ...this.data, host });
  }

  build(): Config {
    return { ...this.data } as Config;
  }
}
```

### **5. Use Phantom Types for Required Fields**

```typescript
// Enforce required fields at compile time
class TypeSafeBuilder<HasId extends boolean = false> {
  setId(id: string): TypeSafeBuilder<true> {
    return this as any;
  }

  build(this: TypeSafeBuilder<true>): Config {
    return {} as Config;
  }
}
```

---

## Summary

### **When to Use Builder Pattern**

- ✅ Many optional parameters
- ✅ Complex construction logic
- ✅ Need validation during construction
- ✅ Want fluent API
- ✅ Building test data

### **When NOT to Use**

- ❌ Simple objects with few fields
- ❌ All fields required
- ❌ Construction is trivial
- ❌ Object rarely changes

### **Key Takeaways**

1. Builders separate construction from representation
2. Method chaining improves readability
3. TypeScript can enforce required fields
4. Validate early and often
5. Return new objects from build()
6. Consider immutability
7. Don't over-engineer simple cases

---

**Interview Questions:**

1. What problem does the Builder pattern solve?
2. How do you enforce required fields at compile time?
3. What's the difference between Builder and Factory patterns?
4. When would you use Builder vs constructor parameters?

**Practice:**

- Build a SQL query builder
- Create a test data builder
- Implement type-safe required fields
- Add validation to a builder
