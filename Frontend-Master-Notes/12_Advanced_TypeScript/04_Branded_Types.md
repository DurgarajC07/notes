# Branded Types (Nominal Typing)

## Core Concept

TypeScript uses structural typing - types are compatible if their structure matches. Branded types add nominal typing to prevent mixing semantically different values with the same structure.

---

## The Problem with Structural Typing

```typescript
type UserId = string;
type ProductId = string;

function getUser(id: UserId) {
  return users.find((u) => u.id === id);
}

const userId: UserId = "user-123";
const productId: ProductId = "product-456";

// ❌ This compiles but is semantically wrong!
getUser(productId); // No error - both are strings
```

---

## Creating Branded Types

```typescript
// Declare unique brand symbol
declare const __brand: unique symbol;

// Branded type with intersection
type Brand<T, TBrand extends string> = T & {
  readonly [__brand]: TBrand;
};

// Create branded types
type UserId = Brand<string, "UserId">;
type ProductId = Brand<string, "ProductId">;

// Smart constructor functions
function createUserId(id: string): UserId {
  // Runtime validation
  if (!id.startsWith("user-")) {
    throw new Error("Invalid user ID format");
  }
  return id as UserId;
}

function createProductId(id: string): ProductId {
  if (!id.startsWith("product-")) {
    throw new Error("Invalid product ID format");
  }
  return id as ProductId;
}

// Usage
const userId = createUserId("user-123");
const productId = createProductId("product-456");

function getUser(id: UserId) {
  return users.find((u) => u.id === id);
}

// ✅ Type safe!
getUser(userId); // ✅ Works
getUser(productId); // ❌ Type error!
getUser("user-123"); // ❌ Type error - must use constructor
```

---

## Numeric Branded Types

```typescript
type Percentage = Brand<number, "Percentage">;
type Pixels = Brand<number, "Pixels">;
type Seconds = Brand<number, "Seconds">;

function toPercentage(n: number): Percentage {
  if (n < 0 || n > 100) {
    throw new Error("Percentage must be 0-100");
  }
  return n as Percentage;
}

function toPixels(n: number): Pixels {
  return n as Pixels;
}

function setOpacity(value: Percentage) {
  element.style.opacity = String(value / 100);
}

function setWidth(value: Pixels) {
  element.style.width = `${value}px`;
}

// ✅ Type safe
setOpacity(toPercentage(50)); // ✅ Works
setWidth(toPixels(200)); // ✅ Works

setOpacity(toPixels(200)); // ❌ Type error!
setWidth(toPercentage(50)); // ❌ Type error!
```

---

## Email and URL Brands

```typescript
type Email = Brand<string, "Email">;
type URL = Brand<string, "URL">;

function createEmail(email: string): Email {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(email)) {
    throw new Error("Invalid email format");
  }
  return email as Email;
}

function createURL(url: string): URL {
  try {
    new window.URL(url);
    return url as URL;
  } catch {
    throw new Error("Invalid URL format");
  }
}

// Usage in API
interface User {
  id: UserId;
  email: Email;
  website: URL;
}

function sendEmail(to: Email, subject: string) {
  // Implementation
}

const user: User = {
  id: createUserId("user-123"),
  email: createEmail("user@example.com"),
  website: createURL("https://example.com"),
};

sendEmail(user.email, "Hello"); // ✅ Works
sendEmail("spam@evil.com", "Spam"); // ❌ Type error!
```

---

## Validated String Brands

```typescript
type NonEmptyString = Brand<string, 'NonEmptyString'>;
type TrimmedString = Brand<string, 'TrimmedString'>;
type SafeHTML = Brand<string, 'SafeHTML'>;

function toNonEmptyString(s: string): NonEmptyString {
  if (s.length === 0) {
    throw new Error('String cannot be empty');
  }
  return s as NonEmptyString;
}

function toTrimmedString(s: string): TrimmedString {
  return s.trim() as TrimmedString;
}

function sanitizeHTML(html: string): SafeHTML {
  // Use DOMPurify or similar
  return DOMPurify.sanitize(html) as SafeHTML;
}

// Component requires non-empty username
interface Props {
  username: NonEmptyString;
  bio: TrimmedString;
  content: SafeHTML;
}

function UserProfile({ username, bio, content }: Props) {
  return (
    <div>
      <h1>{username}</h1>
      <p>{bio}</p>
      <div dangerouslySetInnerHTML={{ __html: content }} />
    </div>
  );
}

// Usage
<UserProfile
  username={toNonEmptyString("john")}
  bio={toTrimmedString("  Software Engineer  ")}
  content={sanitizeHTML("<p>Hello</p>")}
/>
```

---

## Positive Number Brand

```typescript
type PositiveNumber = Brand<number, "PositiveNumber">;
type Integer = Brand<number, "Integer">;
type PositiveInteger = Brand<number, "PositiveInteger">;

function toPositive(n: number): PositiveNumber {
  if (n <= 0) throw new Error("Must be positive");
  return n as PositiveNumber;
}

function toInteger(n: number): Integer {
  if (!Number.isInteger(n)) throw new Error("Must be integer");
  return n as Integer;
}

function toPositiveInteger(n: number): PositiveInteger {
  if (!Number.isInteger(n) || n <= 0) {
    throw new Error("Must be positive integer");
  }
  return n as PositiveInteger;
}

// Usage in pagination
interface PaginationParams {
  page: PositiveInteger;
  pageSize: PositiveInteger;
}

function fetchItems(params: PaginationParams) {
  const offset = (params.page - 1) * params.pageSize;
  // Fetch from API
}

fetchItems({
  page: toPositiveInteger(1),
  pageSize: toPositiveInteger(20),
});
```

---

## UUID Brand

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

function generateUUID(): UUID {
  return crypto.randomUUID() as UUID;
}

// Type-safe database operations
interface Entity {
  id: UUID;
}

function findById(id: UUID): Entity | null {
  return db.find(id);
}

const id = generateUUID();
findById(id); // ✅ Works
findById("invalid-id"); // ❌ Type error!
```

---

## Date Brands

```typescript
type ISODateString = Brand<string, "ISODateString">;
type Timestamp = Brand<number, "Timestamp">;

function toISODateString(date: Date): ISODateString {
  return date.toISOString() as ISODateString;
}

function parseISODate(iso: ISODateString): Date {
  return new Date(iso);
}

function toTimestamp(date: Date): Timestamp {
  return date.getTime() as Timestamp;
}

// API expects ISO strings
interface Event {
  id: UUID;
  createdAt: ISODateString;
  updatedAt: ISODateString;
}

const event: Event = {
  id: generateUUID(),
  createdAt: toISODateString(new Date()),
  updatedAt: toISODateString(new Date()),
};
```

---

## Path Brands

```typescript
type AbsolutePath = Brand<string, "AbsolutePath">;
type RelativePath = Brand<string, "RelativePath">;

function toAbsolutePath(path: string): AbsolutePath {
  if (!path.startsWith("/")) {
    throw new Error("Must be absolute path");
  }
  return path as AbsolutePath;
}

function toRelativePath(path: string): RelativePath {
  if (path.startsWith("/")) {
    throw new Error("Must be relative path");
  }
  return path as RelativePath;
}

function readFile(path: AbsolutePath): string {
  return fs.readFileSync(path, "utf-8");
}

function joinPath(base: AbsolutePath, relative: RelativePath): AbsolutePath {
  return `${base}/${relative}` as AbsolutePath;
}

const base = toAbsolutePath("/home/user");
const file = toRelativePath("file.txt");
const fullPath = joinPath(base, file);
readFile(fullPath);
```

---

## Currency Brands

```typescript
type USD = Brand<number, "USD">;
type EUR = Brand<number, "EUR">;
type JPY = Brand<number, "JPY">;

function toUSD(amount: number): USD {
  return amount as USD;
}

function toEUR(amount: number): EUR {
  return amount as EUR;
}

function convertUSDtoEUR(amount: USD, rate: number): EUR {
  return (amount * rate) as EUR;
}

function charge(amount: USD) {
  stripe.charge({ amount });
}

const price = toUSD(99.99);
charge(price); // ✅ Works

const euroPrice = toEUR(89.99);
charge(euroPrice); // ❌ Type error - wrong currency!

const converted = convertUSDtoEUR(price, 0.85);
```

---

## Combining Brands with Validation

```typescript
// Multiple constraints
type PositiveEvenNumber = Brand<number, "PositiveEvenNumber">;

function toPositiveEven(n: number): PositiveEvenNumber {
  if (n <= 0 || n % 2 !== 0) {
    throw new Error("Must be positive even number");
  }
  return n as PositiveEvenNumber;
}

// Composable brands
type Validated<T> = {
  value: T;
  isValid: boolean;
  errors: string[];
};

function validate<T, B extends string>(
  value: T,
  validator: (v: T) => boolean,
  brand: B,
): Validated<Brand<T, B>> {
  const isValid = validator(value);
  return {
    value: (isValid ? value : null) as any,
    isValid,
    errors: isValid ? [] : [`Invalid ${brand}`],
  };
}
```

---

## Real-World Example: E-commerce

```typescript
type ProductId = Brand<string, "ProductId">;
type OrderId = Brand<string, "OrderId">;
type CustomerId = Brand<string, "CustomerId">;
type Price = Brand<number, "Price">;
type Quantity = Brand<number, "Quantity">;

function createProductId(id: string): ProductId {
  if (!id.startsWith("prod_")) throw new Error("Invalid product ID");
  return id as ProductId;
}

function createOrderId(id: string): OrderId {
  if (!id.startsWith("order_")) throw new Error("Invalid order ID");
  return id as OrderId;
}

function createPrice(amount: number): Price {
  if (amount < 0) throw new Error("Price cannot be negative");
  return (Math.round(amount * 100) / 100) as Price;
}

function createQuantity(qty: number): Quantity {
  if (!Number.isInteger(qty) || qty <= 0) {
    throw new Error("Quantity must be positive integer");
  }
  return qty as Quantity;
}

interface OrderItem {
  productId: ProductId;
  quantity: Quantity;
  price: Price;
}

interface Order {
  id: OrderId;
  customerId: CustomerId;
  items: OrderItem[];
  total: Price;
}

function calculateTotal(items: OrderItem[]): Price {
  const sum = items.reduce((acc, item) => {
    return acc + item.price * item.quantity;
  }, 0);
  return createPrice(sum);
}

function createOrder(customerId: CustomerId, items: OrderItem[]): Order {
  return {
    id: createOrderId(`order_${Date.now()}`),
    customerId,
    items,
    total: calculateTotal(items),
  };
}

// Usage - fully type-safe!
const order = createOrder("cust_123" as CustomerId, [
  {
    productId: createProductId("prod_456"),
    quantity: createQuantity(2),
    price: createPrice(29.99),
  },
]);
```

---

## Best Practices

✅ **Use branded types for IDs** - prevents mixing different entity IDs  
✅ **Validate in constructors** - enforce invariants at creation  
✅ **Brand semantically different values** - even if structurally same  
✅ **Use for API boundaries** - type-safe external data  
✅ **Combine with validation** - runtime safety + compile-time safety  
❌ **Don't overuse** - adds ceremony to code  
❌ **Don't brand internal values** - use for public APIs  
❌ **Don't skip validation** - brand without check is meaningless

---

## Key Takeaways

1. **Branded types add nominal typing** to TypeScript
2. **Prevent mixing semantically different values**
3. **Use smart constructors** for validation
4. **Essential for IDs and validated strings**
5. **Combine with validation** for runtime safety
6. **Great for API boundaries** and public interfaces
7. **Balance type safety with code simplicity**
