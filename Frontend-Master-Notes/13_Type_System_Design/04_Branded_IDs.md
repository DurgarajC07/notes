# 🏷️ Branded IDs - Type-Safe Identifiers

> **Prevent ID Confusion with Nominal Typing in TypeScript's Structural System**

---

## 📋 Table of Contents

- [What are Branded Types?](#what-are-branded-types)
- [The ID Confusion Problem](#the-id-confusion-problem)
- [Basic Branding](#basic-branding)
- [Smart Constructors](#smart-constructors)
- [Multiple Brands](#multiple-brands)
- [Runtime Validation](#runtime-validation)
- [API Integration](#api-integration)
- [Database Operations](#database-operations)
- [Testing](#testing)
- [Best Practices](#best-practices)

---

## What are Branded Types?

Branded types add nominal typing to TypeScript's structural type system, making otherwise compatible types incompatible.

### **The Problem: Structural Typing**

```typescript
type UserId = string;
type ProductId = string;
type OrderId = string;

function getUser(id: UserId): User {
  return db.users.find(id);
}

const productId: ProductId = "prod_123";
const user = getUser(productId); // ❌ No error! But logically wrong
```

### **The Solution: Branding**

```typescript
type UserId = string & { readonly __brand: "UserId" };
type ProductId = string & { readonly __brand: "ProductId" };

function getUser(id: UserId): User {
  return db.users.find(id);
}

const productId: ProductId = "prod_123" as ProductId;
const user = getUser(productId); // ✅ Type error!
// Argument of type 'ProductId' is not assignable to parameter of type 'UserId'
```

---

## The ID Confusion Problem

### **Real-World Scenario**

```typescript
// Without branding - bugs waiting to happen
interface Comment {
  id: string;
  postId: string;
  userId: string;
}

interface Post {
  id: string;
  authorId: string;
}

// Accidentally swap IDs - no compile error!
function badFunction(post: Post, comment: Comment) {
  // Wrong! Using comment ID for post
  const relatedPost = getPost(comment.id);

  // Wrong! Using post ID for user
  const author = getUser(post.id);
}

// ✅ With branding - caught at compile time
type PostId = string & { __brand: "PostId" };
type CommentId = string & { __brand: "CommentId" };
type UserId = string & { __brand: "UserId" };

interface SafeComment {
  id: CommentId;
  postId: PostId;
  userId: UserId;
}

interface SafePost {
  id: PostId;
  authorId: UserId;
}

function safeFunction(post: SafePost, comment: SafeComment) {
  const relatedPost = getPost(comment.id); // ✅ Type error!
  const author = getUser(post.id); // ✅ Type error!

  // Correct usage
  const correctPost = getPost(post.id); // ✅ Works
  const correctAuthor = getUser(post.authorId); // ✅ Works
}
```

---

## Basic Branding

### **Simple Brand Pattern**

```typescript
// Define branded type
type Brand<T, B> = T & { readonly __brand: B };

// Create specific ID types
type UserId = Brand<string, "UserId">;
type ProductId = Brand<string, "ProductId">;
type OrderId = Brand<string, "OrderId">;

// Constructor functions
function createUserId(id: string): UserId {
  return id as UserId;
}

function createProductId(id: string): ProductId {
  return id as ProductId;
}

// Usage
const userId = createUserId("user_123");
const productId = createProductId("prod_456");

// ✅ Type-safe
function updateUser(id: UserId) {
  /* ... */
}
updateUser(userId); // Works

// ❌ Type error
updateUser(productId); // Error: Type 'ProductId' is not assignable to 'UserId'
```

### **Numeric IDs**

```typescript
type UserId = Brand<number, "UserId">;
type PostId = Brand<number, "PostId">;

const userId: UserId = 123 as UserId;
const postId: PostId = 456 as PostId;

// Can't mix them
function getUser(id: UserId): User {
  /* ... */
}
getUser(postId); // ✅ Type error!
```

### **Symbol-Based Branding**

```typescript
// More robust - can't be accidentally duplicated
declare const USER_ID: unique symbol;
declare const PRODUCT_ID: unique symbol;

type UserId = string & { [USER_ID]: true };
type ProductId = string & { [PRODUCT_ID]: true };

function toUserId(id: string): UserId {
  return id as UserId;
}

function toProductId(id: string): ProductId {
  return id as ProductId;
}
```

---

## Smart Constructors

### **With Validation**

```typescript
type UserId = Brand<string, "UserId">;

// Smart constructor with validation
function createUserId(id: string): UserId {
  if (!id.startsWith("user_")) {
    throw new Error("Invalid UserId format");
  }
  if (id.length < 10) {
    throw new Error("UserId too short");
  }
  return id as UserId;
}

// Safe to use
const validId = createUserId("user_123456"); // ✅ Works
const invalidId = createUserId("123"); // ❌ Throws error
```

### **UUID Validation**

```typescript
type UUID = Brand<string, "UUID">;

function createUUID(id: string): UUID {
  const uuidRegex =
    /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

  if (!uuidRegex.test(id)) {
    throw new Error("Invalid UUID format");
  }

  return id as UUID;
}

// Usage
const uuid = createUUID("550e8400-e29b-41d4-a716-446655440000"); // ✅
const bad = createUUID("not-a-uuid"); // ❌ Throws
```

### **Result Type for Safe Construction**

```typescript
type Result<T, E> = { success: true; value: T } | { success: false; error: E };

function tryCreateUserId(id: string): Result<UserId, string> {
  if (!id.startsWith("user_")) {
    return { success: false, error: "Invalid prefix" };
  }

  if (id.length < 10) {
    return { success: false, error: "Too short" };
  }

  return { success: true, value: id as UserId };
}

// Usage
const result = tryCreateUserId("user_123");
if (result.success) {
  const userId = result.value; // UserId type
  updateUser(userId);
} else {
  console.error(result.error);
}
```

---

## Multiple Brands

### **Composite Branding**

```typescript
type Verified = { __verified: true };
type Encrypted = { __encrypted: true };

type VerifiedEmail = string & Verified;
type EncryptedPassword = string & Encrypted;
type VerifiedAndEncrypted = string & Verified & Encrypted;

function verifyEmail(email: string): VerifiedEmail {
  // Verification logic
  return email as VerifiedEmail;
}

function encryptPassword(password: string): EncryptedPassword {
  // Encryption logic
  return password as EncryptedPassword;
}

// Function requires verified email
function sendEmail(email: VerifiedEmail) {
  /* ... */
}

sendEmail("test@example.com"); // ❌ Error: not verified
const verified = verifyEmail("test@example.com");
sendEmail(verified); // ✅ Works
```

### **Hierarchical Brands**

```typescript
type EntityId = string & { __entityId: true };
type UserId = EntityId & { __userId: true };
type AdminId = UserId & { __adminId: true };

// AdminId is a UserId, which is an EntityId
function processEntity(id: EntityId) {
  /* ... */
}
function processUser(id: UserId) {
  /* ... */
}
function processAdmin(id: AdminId) {
  /* ... */
}

const adminId: AdminId = "admin_123" as AdminId;

processEntity(adminId); // ✅ Admin is an Entity
processUser(adminId); // ✅ Admin is a User
processAdmin(adminId); // ✅ Admin is an Admin

const userId: UserId = "user_123" as UserId;
processAdmin(userId); // ❌ Error: User is not Admin
```

---

## Runtime Validation

### **Type Guards**

```typescript
type UserId = Brand<string, "UserId">;

function isUserId(value: unknown): value is UserId {
  return (
    typeof value === "string" && value.startsWith("user_") && value.length >= 10
  );
}

// Usage
function processUser(id: unknown) {
  if (isUserId(id)) {
    // id is UserId here
    updateUser(id); // ✅ Works
  } else {
    throw new Error("Invalid UserId");
  }
}
```

### **Zod Integration**

```typescript
import { z } from "zod";

const UserIdSchema = z
  .string()
  .startsWith("user_")
  .min(10)
  .transform((id) => id as UserId);

type UserId = Brand<string, "UserId">;

// Parse and validate
const result = UserIdSchema.safeParse("user_123456");
if (result.success) {
  const userId: UserId = result.data;
}

// In API endpoint
app.post("/users/:id", (req, res) => {
  const parsed = UserIdSchema.safeParse(req.params.id);

  if (!parsed.success) {
    return res.status(400).json({ error: "Invalid user ID" });
  }

  const userId = parsed.data; // Branded UserId
  const user = getUser(userId);
  res.json(user);
});
```

### **Class-Based Approach**

```typescript
class UserId {
  private readonly __brand!: "UserId";

  private constructor(private readonly value: string) {}

  static create(id: string): UserId {
    if (!id.startsWith("user_")) {
      throw new Error("Invalid UserId");
    }
    const instance = new UserId(id);
    return instance;
  }

  toString(): string {
    return this.value;
  }

  equals(other: UserId): boolean {
    return this.value === other.value;
  }
}

// Usage
const userId = UserId.create("user_123");
console.log(userId.toString()); // "user_123"

// Can't create directly
const badId = new UserId("anything"); // ❌ Constructor is private
```

---

## API Integration

### **Type-Safe API Calls**

```typescript
type UserId = Brand<string, "UserId">;
type PostId = Brand<string, "PostId">;

interface User {
  id: UserId;
  name: string;
  posts: PostId[];
}

interface Post {
  id: PostId;
  authorId: UserId;
  title: string;
}

// API client with branded IDs
class ApiClient {
  async getUser(id: UserId): Promise<User> {
    const response = await fetch(`/api/users/${id}`);
    const data = await response.json();

    return {
      ...data,
      id: data.id as UserId,
      posts: data.posts.map((p: string) => p as PostId),
    };
  }

  async getPost(id: PostId): Promise<Post> {
    const response = await fetch(`/api/posts/${id}`);
    const data = await response.json();

    return {
      ...data,
      id: data.id as PostId,
      authorId: data.authorId as UserId,
    };
  }

  async getUserPosts(userId: UserId): Promise<Post[]> {
    const user = await this.getUser(userId);

    // Type-safe: can only pass PostId to getPost
    return Promise.all(user.posts.map((postId) => this.getPost(postId)));
  }
}

// Usage
const api = new ApiClient();
const userId = "user_123" as UserId;
const posts = await api.getUserPosts(userId);

// ❌ Can't mix IDs
const postId = "post_456" as PostId;
await api.getUserPosts(postId); // Type error!
```

### **GraphQL Integration**

```typescript
import { gql } from "@apollo/client";

type UserId = Brand<string, "UserId">;

const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
      email
    }
  }
`;

function useUser(id: UserId) {
  return useQuery(GET_USER, {
    variables: { id: id as string }, // Cast for GraphQL
  });
}

// Type-safe usage
const userId = "user_123" as UserId;
const { data } = useUser(userId);
```

---

## Database Operations

### **Prisma Integration**

```typescript
import { PrismaClient } from "@prisma/client";

type UserId = Brand<string, "UserId">;
type PostId = Brand<string, "PostId">;

class UserRepository {
  constructor(private db: PrismaClient) {}

  async findById(id: UserId): Promise<User | null> {
    const user = await this.db.user.findUnique({
      where: { id: id as string },
    });

    if (!user) return null;

    return {
      ...user,
      id: user.id as UserId,
    };
  }

  async create(data: Omit<User, "id">): Promise<User> {
    const user = await this.db.user.create({
      data,
    });

    return {
      ...user,
      id: user.id as UserId,
    };
  }

  async getUserPosts(userId: UserId): Promise<Post[]> {
    const posts = await this.db.post.findMany({
      where: { authorId: userId as string },
    });

    return posts.map((post) => ({
      ...post,
      id: post.id as PostId,
      authorId: post.authorId as UserId,
    }));
  }
}
```

### **Type-Safe Queries**

```typescript
type QueryBuilder<T extends string> = {
  where: (id: Brand<string, T>) => QueryBuilder<T>;
  execute: () => Promise<any>;
};

function createQueryBuilder<T extends string>(table: string): QueryBuilder<T> {
  let whereClause: string | null = null;

  return {
    where(id: Brand<string, T>) {
      whereClause = id as string;
      return this;
    },
    async execute() {
      // Execute SQL with whereClause
      return [];
    },
  };
}

// Usage
const userQuery = createQueryBuilder<"UserId">("users");
const userId = "user_123" as UserId;
const users = await userQuery.where(userId).execute();

const productId = "prod_456" as ProductId;
await userQuery.where(productId); // ❌ Type error!
```

---

## Testing

### **Test Fixtures**

```typescript
type UserId = Brand<string, "UserId">;
type PostId = Brand<string, "PostId">;

// Test helper functions
export const TestIds = {
  user: (id: string = "user_test") => id as UserId,
  post: (id: string = "post_test") => id as PostId,
};

// Usage in tests
describe("User tests", () => {
  it("should get user by ID", async () => {
    const userId = TestIds.user("user_123");
    const user = await getUser(userId);
    expect(user.id).toBe(userId);
  });

  it("should not accept wrong ID type", () => {
    const postId = TestIds.post("post_456");

    // @ts-expect-error - testing type safety
    getUser(postId);
  });
});
```

### **Mock Data**

```typescript
function createMockUser(overrides?: Partial<User>): User {
  return {
    id: TestIds.user(),
    name: "Test User",
    email: "test@example.com",
    ...overrides,
  };
}

function createMockPost(overrides?: Partial<Post>): Post {
  return {
    id: TestIds.post(),
    authorId: TestIds.user(),
    title: "Test Post",
    ...overrides,
  };
}

// Usage
const user = createMockUser({ name: "John" });
const post = createMockPost({ authorId: user.id }); // Type-safe!
```

---

## Best Practices

### **1. Use Symbol Brands for Security**

```typescript
// ✅ Good: Symbol prevents accidental brand duplication
declare const USER_ID: unique symbol;
type UserId = string & { [USER_ID]: true };

// ❌ Bad: String brands can be duplicated
type UserId = string & { __brand: "UserId" };
```

### **2. Always Use Smart Constructors**

```typescript
// ✅ Good: Validation at construction
function createUserId(id: string): UserId {
  if (!isValidUserId(id)) {
    throw new Error("Invalid UserId");
  }
  return id as UserId;
}

// ❌ Bad: Raw casting everywhere
const userId = "anything" as UserId;
```

### **3. Document Brand Semantics**

```typescript
/**
 * Unique identifier for a user.
 * Format: "user_" followed by 24 alphanumeric characters
 * Example: "user_5f9d88a9c5d3e8a6b4c7d9e1"
 */
type UserId = Brand<string, "UserId">;
```

### **4. Use Type Guards for Unknown Data**

```typescript
function isUserId(value: unknown): value is UserId {
  return typeof value === "string" && value.startsWith("user_");
}

// ✅ Safe handling of external data
function processExternalData(data: unknown) {
  if (isUserId(data)) {
    updateUser(data);
  }
}
```

### **5. Consider Performance**

```typescript
// Brands have zero runtime cost
type UserId = Brand<string, "UserId">;

// At runtime, UserId is just a string
const userId: UserId = "user_123" as UserId;
console.log(typeof userId); // "string"
```

---

## Summary

### **Benefits**

- ✅ Prevent ID confusion at compile time
- ✅ Zero runtime overhead
- ✅ Self-documenting code
- ✅ Catch bugs early
- ✅ Better IDE support

### **When to Use**

- Multiple ID types (User, Post, Order, etc.)
- API boundaries
- Database operations
- Security-sensitive data
- Complex domain models

### **Implementation Checklist**

- [ ] Define branded types
- [ ] Create smart constructors
- [ ] Add validation
- [ ] Implement type guards
- [ ] Document format requirements
- [ ] Create test helpers
- [ ] Use consistently across codebase

---

**Interview Questions:**

1. What problem do branded types solve?
2. How do branded types work with TypeScript's structural typing?
3. What's the runtime cost of branded types?
4. How do you validate branded types from external sources?

**Practice:**

- Brand all IDs in an existing codebase
- Create smart constructors with validation
- Implement type guards for runtime checks
- Build a type-safe API client
