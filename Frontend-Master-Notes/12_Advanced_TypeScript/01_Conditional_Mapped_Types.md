# 🎨 Advanced TypeScript Patterns

> Type-level programming, conditional types, and template literal types unlock TypeScript's full potential. These patterns enable type-safe APIs, better inference, and self-documenting code.

---

## 📖 1. Concept Explanation

### Conditional Types

Conditional types select types based on conditions (similar to ternary operators for types):

```typescript
// Basic conditional type
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false

// Extract function return type
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type Func = () => string;
type R = ReturnType<Func>; // string
```

### Mapped Types

Transform properties of existing types:

```typescript
// Make all properties optional
type Partial<T> = {
  [P in keyof T]?: T[P];
};

// Make all properties readonly
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};

// Pick specific properties
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};
```

### Template Literal Types

Build string types from other types:

```typescript
// Event names
type Events = "click" | "focus" | "blur";
type EventHandlers = `on${Capitalize<Events>}`;
// 'onClick' | 'onFocus' | 'onBlur'

// API endpoints
type HttpMethod = "get" | "post" | "put" | "delete";
type Endpoint = `/api/users`;
type ApiRoute = `${HttpMethod} ${Endpoint}`;
// 'get /api/users' | 'post /api/users' | ...
```

### Utility Types (Built-in)

```typescript
// Partial<T> - All properties optional
type User = { name: string; age: number };
type PartialUser = Partial<User>; // { name?: string; age?: number; }

// Required<T> - All properties required
type RequiredUser = Required<PartialUser>; // { name: string; age: number; }

// Pick<T, K> - Select properties
type NameOnly = Pick<User, "name">; // { name: string; }

// Omit<T, K> - Exclude properties
type WithoutAge = Omit<User, "age">; // { name: string; }

// Record<K, T> - Map keys to type
type Roles = "admin" | "user" | "guest";
type Permissions = Record<Roles, string[]>;
// { admin: string[]; user: string[]; guest: string[]; }

// Exclude<T, U> - Remove types
type T = Exclude<"a" | "b" | "c", "a">; // 'b' | 'c'

// Extract<T, U> - Extract types
type E = Extract<"a" | "b" | "c", "a" | "d">; // 'a'

// NonNullable<T> - Remove null/undefined
type T2 = NonNullable<string | null | undefined>; // string

// ReturnType<T> - Extract return type
type R2 = ReturnType<() => string>; // string

// Parameters<T> - Extract parameter types
type P = Parameters<(a: string, b: number) => void>; // [string, number]
```

---

## 🧠 2. Why It Matters in Real Projects

### Type-Safe APIs

```typescript
// Without advanced types: Manual typing, error-prone
interface User {
  id: number;
  name: string;
  email: string;
}

function updateUser(id: number, updates: any) {
  // ❌ any!
  // ...
}

// With advanced types: Type-safe partial updates
function updateUser(id: number, updates: Partial<User>) {
  // TS knows: updates.id is number | undefined
  // TS knows: updates.name is string | undefined
  // TS prevents: updates.age (not in User)
}
```

### React Component Props

```typescript
// Extract prop types from component
type ButtonProps = React.ComponentProps<"button">;
// Contains all standard button props + event handlers

type MyButtonProps = ButtonProps & {
  variant: "primary" | "secondary";
  loading?: boolean;
};

// Omit conflicting props
type CustomButtonProps = Omit<ButtonProps, "type"> & {
  type: "submit" | "reset"; // Narrower than HTML button type
};
```

### State Management

```typescript
// Redux-style type-safe actions
type Action<T = any> = {
  type: string;
  payload: T;
};

// Discriminated union for all actions
type AppAction =
  | { type: "user/login"; payload: { email: string } }
  | { type: "user/logout" }
  | { type: "todos/add"; payload: { text: string } };

function reducer(state: State, action: AppAction) {
  switch (action.type) {
    case "user/login":
      // TS knows: action.payload is { email: string; }
      return { ...state, user: action.payload };

    case "user/logout":
      // TS knows: no payload
      return { ...state, user: null };

    case "todos/add":
      // TS knows: action.payload is { text: string; }
      return { ...state, todos: [...state.todos, action.payload] };
  }
}
```

---

## ⚙️ 3. Internal Working

### How Conditional Types Work

TypeScript evaluates conditional types during type checking:

```typescript
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// When TS sees: MyReturnType<() => string>
// 1. Check: Does "() => string" extend "(...args: any[]) => infer R"?
// 2. Yes! Match and infer R = string
// 3. Return: R (which is string)

// When TS sees: MyReturnType<number>
// 1. Check: Does "number" extend "(...args: any[]) => infer R"?
// 2. No!
// 3. Return: never
```

### Distributive Conditional Types

Conditional types distribute over unions:

```typescript
type ToArray<T> = T extends any ? T[] : never;

type Result = ToArray<string | number>;
// Distributes to: ToArray<string> | ToArray<number>
// Evaluates to: string[] | number[]

// Non-distributive (use [T] to prevent distribution)
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;

type Result2 = ToArrayNonDist<string | number>;
// Evaluates to: (string | number)[]
```

### Template Literal Type Inference

```typescript
type ExtractPathParams<T extends string> =
  T extends `${infer Start}/:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof ExtractPathParams<`/${Rest}`>]: string }
    : T extends `${infer Start}/:${infer Param}`
      ? { [K in Param]: string }
      : {};

type Params = ExtractPathParams<"/users/:id/posts/:postId">;
// { id: string; postId: string; }

// TS parses template literal recursively:
// 1. Match "/users/:id/posts/:postId"
// 2. Extract "id" from first /:id/
// 3. Recurse on "/posts/:postId"
// 4. Extract "postId"
// 5. Combine: { id: string; postId: string; }
```

---

## ✅ 4. Best Practices

### DO ✅

```typescript
// 1. Use const assertions for literal types
const ROLES = ['admin', 'user', 'guest'] as const;
type Role = typeof ROLES[number]; // 'admin' | 'user' | 'guest'

// 2. Branded types for type safety
type UserId = string & { readonly __brand: 'UserId' };
type ProductId = string & { readonly __brand: 'ProductId' };

function getUser(id: UserId) { /* ... */ }
function getProduct(id: ProductId) { /* ... */ }

// ❌ Won't compile:
const userId: UserId = 'user-123' as UserId;
const productId: ProductId = 'prod-456' as ProductId;
getUser(productId); // Error: Type 'ProductId' not assignable to 'UserId'

// 3. Use generics with constraints
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach(key => {
    result[key] = obj[key];
  });
  return result;
}

// 4. Discriminated unions for state
type LoadingState = { status: 'loading' };
type SuccessState<T> = { status: 'success'; data: T };
type ErrorState = { status: 'error'; error: Error };

type AsyncState<T> = LoadingState | SuccessState<T> | ErrorState;

function render<T>(state: AsyncState<T>) {
  switch (state.status) {
    case 'loading':
      return <Spinner />;

    case 'success':
      return <Data data={state.data} />; // TS knows: state.data exists

    case 'error':
      return <Error error={state.error} />; // TS knows: state.error exists
  }
}

// 5. Recursive types for nested structures
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

interface Config {
  api: {
    baseUrl: string;
    timeout: number;
  };
  ui: {
    theme: string;
  };
}

const partialConfig: DeepPartial<Config> = {
  api: {
    timeout: 5000 // baseUrl omitted (optional)
  }
  // ui omitted (optional)
};
```

### DON'T ❌

```typescript
// 1. Don't overuse any
function process(data: any) {
  // ❌ No type safety
  return data.value;
}

// ✅ Use generics
function process<T extends { value: unknown }>(data: T) {
  return data.value;
}

// 2. Don't use type assertions unnecessarily
const user = getUser() as User; // ❌ Unsafe

// ✅ Use type guards
const user = getUser();
if (isUser(user)) {
  // TS knows: user is User
}

// 3. Don't create overly complex types
// ❌ Unreadable
type ComplexType<T> = T extends (infer U)[]
  ? U extends object
    ? { [K in keyof U]: U[K] extends Function ? never : U[K] }[]
    : T
  : never;

// ✅ Break into smaller types
type ExtractArrayItem<T> = T extends (infer U)[] ? U : never;
type FilterFunctions<T> = {
  [K in keyof T]: T[K] extends Function ? never : T[K];
};
type SimplerType<T> = FilterFunctions<ExtractArrayItem<T>>[];

// 4. Don't ignore type errors with @ts-ignore
// @ts-ignore  // ❌
user.nonExistentProp = "value";

// ✅ Fix the type
interface ExtendedUser extends User {
  nonExistentProp: string;
}

// 5. Don't abuse union types
type StringOrNumberOrBooleanOrObjectOrArray =
  | string
  | number
  | boolean
  | object
  | any[]; // ❌ Too wide

// ✅ Be specific
type PrimitiveValue = string | number | boolean;
type ComplexValue = { [key: string]: PrimitiveValue };
type ConfigValue = PrimitiveValue | ComplexValue;
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Not Understanding Type Narrowing

```typescript
// ❌ Type assertion instead of narrowing
function processValue(value: string | number) {
  const str = value as string; // Unsafe!
  return str.toUpperCase();
}

// ✅ Type guard
function processValue(value: string | number) {
  if (typeof value === "string") {
    return value.toUpperCase(); // TS knows: value is string
  }
  return value.toString();
}
```

### Mistake #2: Mutable Arrays Breaking Literal Types

```typescript
// ❌ Array is mutable
const ROLES = ["admin", "user"];
type Role = (typeof ROLES)[number]; // string (too wide!)

// ✅ Use const assertion
const ROLES = ["admin", "user"] as const;
type Role = (typeof ROLES)[number]; // 'admin' | 'user'
```

### Mistake #3: Incorrect Generic Constraints

```typescript
// ❌ Too restrictive
function merge<T extends object, U extends object>(a: T, b: U) {
  return { ...a, ...b };
}

merge({ a: 1 }, { b: 2 }); // Works
merge({ a: 1 }, "string"); // Error (good)
merge(null, { b: 2 }); // Error (null is object in JS, but not in TS constraint)

// ✅ Better constraint
function merge<
  T extends Record<string, unknown>,
  U extends Record<string, unknown>,
>(a: T, b: U): T & U {
  return { ...a, ...b };
}
```

### Mistake #4: Not Using Discriminated Unions

```typescript
// ❌ Hard to narrow
interface Success {
  data: any;
  error?: never;
}

interface Error {
  error: string;
  data?: never;
}

type Result = Success | Error;

function handle(result: Result) {
  if (result.data) {
    // Not reliable
    // ...
  }
}

// ✅ Discriminated union
interface Success {
  status: "success";
  data: any;
}

interface Error {
  status: "error";
  error: string;
}

type Result = Success | Error;

function handle(result: Result) {
  if (result.status === "success") {
    console.log(result.data); // TS knows: result.data exists
  } else {
    console.log(result.error); // TS knows: result.error exists
  }
}
```

---

## 🔐 6. Security Considerations

### Prevent Type Confusion Attacks

```typescript
// ❌ UNSAFE: User input treated as safe
type UserId = string;
type SQL = string;

function deleteUser(userId: UserId) {
  const query: SQL = `DELETE FROM users WHERE id = ${userId}`; // SQL injection!
}

// ✅ SAFE: Branded types prevent mixing
type UserId = string & { readonly __brand: "UserId" };
type SQL = string & { readonly __brand: "SQL" };

function deleteUser(userId: UserId) {
  const query: SQL = sql`DELETE FROM users WHERE id = ${userId}`;
  // sql template tag sanitizes and brands as SQL
}

// Helper to create branded SQL
function sql(strings: TemplateStringsArray, ...values: unknown[]): SQL {
  // Sanitize values
  const sanitized = values.map((v) => sanitize(v));
  return strings.reduce(
    (acc, str, i) => acc + str + (sanitized[i] || ""),
    "",
  ) as SQL;
}
```

### Validate External Data

```typescript
// ❌ Trust external data
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  return response.json(); // Assumes correct shape!
}

// ✅ Runtime validation with type guard
function isUser(obj: unknown): obj is User {
  return (
    typeof obj === "object" &&
    obj !== null &&
    "id" in obj &&
    "name" in obj &&
    "email" in obj &&
    typeof obj.id === "string" &&
    typeof obj.name === "string" &&
    typeof obj.email === "string"
  );
}

async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data: unknown = await response.json();

  if (isUser(data)) {
    return data;
  }

  throw new Error("Invalid user data");
}

// Or use validation libraries: zod, yup, io-ts
import { z } from "zod";

const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email(),
});

type User = z.infer<typeof UserSchema>;

async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();
  return UserSchema.parse(data); // Throws if invalid
}
```

---

## 🚀 7. Performance Optimization

### 1. Avoid Complex Recursive Types

```typescript
// ❌ SLOW: Deep recursion
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

// On large nested objects, TS can be slow

// ✅ FASTER: Limit recursion depth
type DeepReadonly<T, Depth extends number = 5> = Depth extends 0
  ? T
  : {
      readonly [P in keyof T]: T[P] extends object
        ? DeepReadonly<T[P], Prev<Depth>>
        : T[P];
    };

type Prev<N extends number> = [-1, 0, 1, 2, 3, 4, 5][N];
```

### 2. Use Type Aliases Over Interfaces for Unions

```typescript
// ❌ SLOWER: Interface with union
interface Result {
  type: "success" | "error";
  data?: any;
  error?: string;
}

// ✅ FASTER: Discriminated union with type alias
type Result = { type: "success"; data: any } | { type: "error"; error: string };
```

### 3. Memoize Expensive Type Computations

```typescript
// TypeScript caches type computations, but can help it:

// ❌ Recomputed each time
function process<T>(data: ComplexTransform<ComplexTransform<T>>) {
  // ...
}

// ✅ Extract to type alias (cached)
type DoubleTransform<T> = ComplexTransform<ComplexTransform<T>>;

function process<T>(data: DoubleTransform<T>) {
  // ...
}
```

---

## 🧪 8. Code Examples

### Example 1: Type-Safe Event Emitter

```typescript
type EventMap = {
  "user:login": { userId: string; timestamp: number };
  "user:logout": { userId: string };
  "data:update": { data: any[] };
};

class TypedEventEmitter<T extends Record<string, any>> {
  private listeners: {
    [K in keyof T]?: Array<(payload: T[K]) => void>;
  } = {};

  on<K extends keyof T>(event: K, handler: (payload: T[K]) => void) {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event]!.push(handler);
  }

  emit<K extends keyof T>(event: K, payload: T[K]) {
    this.listeners[event]?.forEach((handler) => handler(payload));
  }
}

const emitter = new TypedEventEmitter<EventMap>();

emitter.on("user:login", (payload) => {
  // TS knows: payload is { userId: string; timestamp: number }
  console.log(payload.userId, payload.timestamp);
});

emitter.emit("user:login", {
  userId: "123",
  timestamp: Date.now(),
}); // ✅

emitter.emit("user:login", { userId: "123" }); // ❌ Error: missing timestamp
```

### Example 2: Type-Safe Route Params

```typescript
type ExtractParams<T extends string> =
  T extends `${infer _Start}:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof ExtractParams<Rest>]: string }
    : T extends `${infer _Start}:${infer Param}`
      ? { [K in Param]: string }
      : {};

function route<T extends string>(
  path: T,
  handler: (params: ExtractParams<T>) => void,
) {
  // Router logic
}

route("/users/:userId/posts/:postId", (params) => {
  // TS knows: params is { userId: string; postId: string }
  console.log(params.userId, params.postId);
  console.log(params.invalid); // ❌ Error
});

route("/api/data", (params) => {
  // TS knows: params is {}
  console.log(params); // {}
});
```

### Example 3: Chainable Builder Pattern

```typescript
type QueryBuilder<T, Selected = never> = {
  select<K extends keyof T>(...keys: K[]): QueryBuilder<T, Selected | K>;

  where(filter: Partial<T>): QueryBuilder<T, Selected>;

  execute(): Promise<Selected extends keyof T ? Pick<T, Selected>[] : T[]>;
};

// Usage
interface User {
  id: string;
  name: string;
  email: string;
  age: number;
}

const query: QueryBuilder<User> = createQuery();

const result = await query
  .select("id", "name") // Returns QueryBuilder<User, 'id' | 'name'>
  .where({ age: 25 })
  .execute();

// TS knows: result is { id: string; name: string }[]
result[0].id; // ✅
result[0].email; // ❌ Error
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: Type-Safe Redux

```typescript
// Actions
type Action<T extends string, P = void> = P extends void
  ? { type: T }
  : { type: T; payload: P };

type AppAction =
  | Action<"user/login", { email: string; token: string }>
  | Action<"user/logout">
  | Action<"todos/add", { text: string }>
  | Action<"todos/toggle", { id: string }>;

// Action creators
function createAction<T extends AppAction["type"]>(
  type: T,
): Extract<AppAction, { type: T }> extends { payload: infer P }
  ? (payload: P) => { type: T; payload: P }
  : () => { type: T };

const login = createAction("user/login");
const logout = createAction("user/logout");

login({ email: "a@b.com", token: "xyz" }); // ✅
logout(); // ✅
login(); // ❌ Error: missing payload
```

### Use Case 2: Type-Safe API Client

```typescript
type ApiRoutes = {
  "GET /users": { response: User[] };
  "GET /users/:id": { params: { id: string }; response: User };
  "POST /users": { body: CreateUserDto; response: User };
  "PUT /users/:id": {
    params: { id: string };
    body: UpdateUserDto;
    response: User;
  };
};

type ExtractMethod<T extends string> = T extends `${infer M} ${string}`
  ? M
  : never;
type ExtractPath<T extends string> = T extends `${string} ${infer P}`
  ? P
  : never;

class ApiClient<Routes extends Record<string, any>> {
  async request<
    Route extends keyof Routes,
    Method extends string = ExtractMethod<Route & string>,
    Path extends string = ExtractPath<Route & string>,
  >(
    route: Route,
    options?: Routes[Route] extends { params: infer P }
      ? { params: P }
      : {} & Routes[Route] extends { body: infer B }
        ? { body: B }
        : {},
  ): Promise<Routes[Route]["response"]> {
    // Fetch logic
    return {} as Routes[Route]["response"];
  }
}

const api = new ApiClient<ApiRoutes>();

// All type-safe!
await api.request("GET /users"); // Returns Promise<User[]>
await api.request("GET /users/:id", {
  params: { id: "123" },
}); // Returns Promise<User>

await api.request("POST /users", {
  body: { name: "Alice", email: "alice@example.com" },
}); // Returns Promise<User>
```

---

## ❓ 10. Interview Questions

### Q1: What are conditional types and when would you use them?

**Answer:**

Conditional types select types based on conditions, similar to ternary operators:

```typescript
type IsString<T> = T extends string ? true : false;
```

**When to use:**

1. **Extract types:**

```typescript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
```

2. **Type transformations:**

```typescript
type Nullable<T> = T extends null | undefined ? never : T;
```

3. **Distributive types:**

```typescript
type ToArray<T> = T extends any ? T[] : never;
type Result = ToArray<string | number>; // string[] | number[]
```

4. **Type guards:**

```typescript
type NonNullable<T> = T extends null | undefined ? never : T;
```

**Real-world example:**

```typescript
// Extract async function return type
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;

async function fetchUser(): Promise<User> {}

type U = UnwrapPromise<ReturnType<typeof fetchUser>>; // User
```

---

### Q2: Explain mapped types and provide examples.

**Answer:**

Mapped types transform properties of existing types:

```typescript
type Mapped<T> = {
  [P in keyof T]: NewType;
};
```

**Examples:**

```typescript
// 1. Make all properties optional
type Partial<T> = {
  [P in keyof T]?: T[P];
};

// 2. Make all properties readonly
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};

// 3. Add null to all properties
type Nullable<T> = {
  [P in keyof T]: T[P] | null;
};

// 4. Transform property types
type Getters<T> = {
  [P in keyof T as `get${Capitalize<string & P>}`]: () => T[P];
};

interface User {
  name: string;
  age: number;
}

type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number; }
```

**Real-world use:**

```typescript
// Convert all methods to async
type Asyncify<T> = {
  [P in keyof T]: T[P] extends (...args: infer A) => infer R
    ? (...args: A) => Promise<R>
    : T[P];
};

interface SyncAPI {
  getUser(id: string): User;
  updateUser(id: string, data: Partial<User>): User;
}

type AsyncAPI = Asyncify<SyncAPI>;
// {
//   getUser(id: string): Promise<User>;
//   updateUser(id: string, data: Partial<User>): Promise<User>;
// }
```

---

### Q3: How do template literal types work?

**Answer:**

Template literal types build string literal types from other types:

```typescript
type Greeting = `Hello, ${string}`;
type Specific = `Hello, ${"World" | "TypeScript"}`;
// 'Hello, World' | 'Hello, TypeScript'
```

**Use cases:**

1. **Event names:**

```typescript
type Events = "click" | "focus" | "blur";
type Handlers = `on${Capitalize<Events>}`;
// 'onClick' | 'onFocus' | 'onBlur'
```

2. **API routes:**

```typescript
type HttpMethod = "GET" | "POST";
type Endpoint = "/users" | "/posts";
type Route = `${HttpMethod} ${Endpoint}`;
// 'GET /users' | 'GET /posts' | 'POST /users' | 'POST /posts'
```

3. **Extract path params:**

```typescript
type ExtractParams<T extends string> =
  T extends `${infer _}:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof ExtractParams<Rest>]: string }
    : T extends `${infer _}:${infer Param}`
      ? { [K in Param]: string }
      : {};

type Params = ExtractParams<"/users/:userId/posts/:postId">;
// { userId: string; postId: string }
```

---

### Q4: What are branded types and why use them?

**Answer:**

Branded types (nominal typing) prevent accidentally mixing similar types:

```typescript
type Brand<K, T> = K & { __brand: T };

type UserId = Brand<string, "UserId">;
type ProductId = Brand<string, "ProductId">;

function getUser(id: UserId) {}
function getProduct(id: ProductId) {}

const userId = "user-123" as UserId;
const productId = "prod-456" as ProductId;

getUser(userId); // ✅
getUser(productId); // ❌ Error: Type 'ProductId' not assignable to 'UserId'
```

**Why use them:**

1. **Prevent confusion:**

```typescript
// Without branding:
function transfer(from: string, to: string, amount: number) {
  // Easy to swap 'from' and 'to'
}

// With branding:
type AccountId = Brand<string, "AccountId">;
function transfer(from: AccountId, to: AccountId, amount: number) {
  // Type-safe, can't swap
}
```

2. **Validated values:**

```typescript
type Email = Brand<string, "Email">;

function validateEmail(input: string): Email | null {
  if (/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(input)) {
    return input as Email;
  }
  return null;
}

function sendEmail(to: Email) {
  // Guaranteed valid email
}

sendEmail("invalid"); // ❌ Error
const email = validateEmail("valid@example.com");
if (email) {
  sendEmail(email); // ✅
}
```

---

### Q5: How do you create a type-safe deep partial?

**Answer:**

Deep partial recursively makes all properties optional:

```typescript
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

interface Config {
  server: {
    host: string;
    port: number;
    ssl: {
      enabled: boolean;
      cert: string;
    };
  };
  database: {
    url: string;
  };
}

const partialConfig: DeepPartial<Config> = {
  server: {
    port: 3000,
    ssl: {
      enabled: true,
      // cert omitted (optional)
    },
    // host omitted (optional)
  },
  // database omitted (optional)
};
```

**Improved version (handles arrays, primitives):**

```typescript
type DeepPartial<T> = T extends Function
  ? T
  : T extends object
    ? T extends Array<infer U>
      ? Array<DeepPartial<U>>
      : { [P in keyof T]?: DeepPartial<T[P]> }
    : T;
```

---

## 🧩 11. Design Patterns

### Pattern 1: Builder with Fluent API

```typescript
class QueryBuilder<T, Selected extends keyof T = never> {
  private _selected: Set<keyof T> = new Set();
  private _filters: Partial<T> = {};

  select<K extends keyof T>(...keys: K[]): QueryBuilder<T, Selected | K> {
    keys.forEach((k) => this._selected.add(k));
    return this as unknown as QueryBuilder<T, Selected | K>;
  }

  where(filter: Partial<T>): this {
    this._filters = { ...this._filters, ...filter };
    return this;
  }

  execute(): Promise<Pick<T, Selected>[]> {
    // Query logic
    return Promise.resolve([]);
  }
}

const result = await new QueryBuilder<User>()
  .select("id", "name")
  .where({ age: 25 })
  .execute();

// TS knows: result is { id: string; name: string }[]
```

### Pattern 2: Discriminated Union for State

```typescript
type AsyncState<T, E = Error> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: E };

function render<T>(state: AsyncState<T>) {
  switch (state.status) {
    case 'idle':
      return <div>Click to load</div>;

    case 'loading':
      return <Spinner />;

    case 'success':
      return <Data data={state.data} />;

    case 'error':
      return <Error error={state.error} />;
  }
}
```

---

## 🎯 Summary

Advanced TypeScript patterns enable:

- **Type safety** - Catch errors at compile time
- **Better IntelliSense** - IDE autocomplete and hints
- **Self-documenting code** - Types explain intent
- **Refactoring confidence** - TS finds breaking changes

**Key techniques:**

- Conditional types (type-level ternary)
- Mapped types (transform object types)
- Template literals (build string types)
- Branded types (nominal typing)
- Utility types (Partial, Pick, Omit, etc.)

**Production benefits:**

- Fewer runtime errors
- Better developer experience
- Easier onboarding (types as documentation)
- Safer refactoring

Master these patterns to build robust, maintainable TypeScript applications that scale.
