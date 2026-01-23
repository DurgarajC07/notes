# 🔷 TypeScript Interview Questions (Senior Level)

> Senior TypeScript interview questions covering type system, generics, advanced patterns, and practical application. Essential for Staff-level frontend roles.

---

## 🎯 Part 1: Type System Fundamentals

### Q1: What's the difference between `type` and `interface`?

**Answer:**

Both define object shapes, but have subtle differences:

| `type`                                              | `interface`                   |
| --------------------------------------------------- | ----------------------------- |
| Can represent any type (primitives, unions, tuples) | Only object shapes            |
| Can't be reopened (declaration merging)             | Can be reopened               |
| Better for unions/intersections                     | Better for extension          |
| Can use computed properties                         | Can't use computed properties |

**Examples:**

```typescript
// ✅ type: Can represent primitives
type ID = string | number;
type Status = 'pending' | 'active' | 'inactive';

// ❌ interface: Can't represent primitives
interface ID = string | number; // ERROR

// ✅ interface: Declaration merging
interface User {
  name: string;
}

interface User {
  age: number;
}

// Result: User = { name: string; age: number; }

// ❌ type: Can't be reopened
type User = { name: string };
type User = { age: number }; // ERROR: Duplicate identifier

// ✅ type: Union types
type Result = Success | Error;

// ❌ interface: Can't directly represent unions
interface Result = Success | Error; // ERROR

// ✅ Both: Extension
interface Animal {
  name: string;
}

interface Dog extends Animal {
  breed: string;
}

type Animal = {
  name: string;
};

type Dog = Animal & {
  breed: string;
};
```

**Real-world decision:**

```typescript
// ✅ Use interface for public APIs (extendable)
export interface ApiResponse {
  data: unknown;
  status: number;
}

// Libraries can extend
declare module "my-lib" {
  interface ApiResponse {
    timestamp: Date;
  }
}

// ✅ Use type for internal utilities
type Nullable<T> = T | null;
type DeepPartial<T> = { [K in keyof T]?: DeepPartial<T[K]> };
```

**Production rule:** Use `interface` for object shapes (especially public APIs), `type` for unions, tuples, and utility types.

---

### Q2: Explain structural vs nominal typing

**Answer:**

**Structural typing (TypeScript):** Types are compatible if their structure matches

```typescript
interface Point2D {
  x: number;
  y: number;
}

interface Vector {
  x: number;
  y: number;
}

// ✅ Compatible! Same structure
const point: Point2D = { x: 1, y: 2 };
const vector: Vector = point; // No error

// This can cause issues:
type UserId = number;
type ProductId = number;

function getUser(id: UserId) {
  /* ... */
}

const productId: ProductId = 123;
getUser(productId); // ✅ No error (both are numbers)
```

**Nominal typing (simulated with branded types):**

```typescript
// ✅ Brand types to prevent mixing
type UserId = number & { __brand: "UserId" };
type ProductId = number & { __brand: "ProductId" };

function createUserId(id: number): UserId {
  return id as UserId;
}

function createProductId(id: number): ProductId {
  return id as ProductId;
}

function getUser(id: UserId) {
  /* ... */
}

const userId = createUserId(1);
const productId = createProductId(123);

getUser(userId); // ✅ OK
getUser(productId); // ❌ ERROR: Type 'ProductId' not assignable to 'UserId'
```

**Real-world application:**

```typescript
// Banking system
type AccountNumber = string & { __brand: "AccountNumber" };
type RoutingNumber = string & { __brand: "RoutingNumber" };

function transfer(from: AccountNumber, to: AccountNumber, amount: number) {
  // Can't accidentally pass RoutingNumber
}
```

---

### Q3: How do generics work? Provide real-world examples.

**Answer:**

**Generics** allow type-safe reusable code without losing type information.

**Basic generic:**

```typescript
// ❌ Without generics: Loses type info
function identity(value: any): any {
  return value;
}

const result = identity("hello"); // Type: any

// ✅ With generics: Preserves type
function identity<T>(value: T): T {
  return value;
}

const result = identity("hello"); // Type: string
const num = identity(42); // Type: number
```

**Generic constraints:**

```typescript
// Constraint: T must have length property
function getLength<T extends { length: number }>(item: T): number {
  return item.length;
}

getLength("hello"); // ✅ string has length
getLength([1, 2, 3]); // ✅ array has length
getLength(42); // ❌ ERROR: number doesn't have length
```

**Real-world example 1: API wrapper**

```typescript
interface ApiResponse<T> {
  data: T;
  status: number;
  error?: string;
}

async function fetchData<T>(url: string): Promise<ApiResponse<T>> {
  const response = await fetch(url);
  const data = await response.json();

  return {
    data,
    status: response.status,
  };
}

// Usage: Type is inferred
interface User {
  id: number;
  name: string;
}

const result = await fetchData<User>("/api/users/1");
// result.data is typed as User!

console.log(result.data.name); // ✅ TypeScript knows this is string
```

**Real-world example 2: Type-safe event emitter**

```typescript
type EventMap = {
  "user:login": { userId: string; timestamp: Date };
  "user:logout": { userId: string };
  "payment:success": { amount: number; orderId: string };
};

class TypedEventEmitter<T extends Record<string, any>> {
  private listeners: {
    [K in keyof T]?: Array<(data: T[K]) => void>;
  } = {};

  on<K extends keyof T>(event: K, callback: (data: T[K]) => void) {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event]!.push(callback);
  }

  emit<K extends keyof T>(event: K, data: T[K]) {
    const callbacks = this.listeners[event] || [];
    callbacks.forEach((callback) => callback(data));
  }
}

// Usage
const emitter = new TypedEventEmitter<EventMap>();

// ✅ Type-safe: data is { userId: string; timestamp: Date }
emitter.on("user:login", (data) => {
  console.log(data.userId); // ✅ TypeScript knows this exists
  console.log(data.timestamp); // ✅ TypeScript knows this is Date
});

// ✅ Type-safe emit
emitter.emit("user:login", {
  userId: "123",
  timestamp: new Date(),
});

// ❌ ERROR: Missing required property
emitter.emit("user:login", { userId: "123" }); // ERROR: timestamp missing

// ❌ ERROR: Wrong event name
emitter.on("user:signin", (data) => {}); // ERROR: 'user:signin' doesn't exist
```

**Real-world example 3: Type-safe form builder**

```typescript
type FormConfig<T> = {
  [K in keyof T]: {
    label: string;
    type: "text" | "number" | "email" | "password";
    required?: boolean;
    validate?: (value: T[K]) => string | undefined;
  };
};

function createForm<T extends Record<string, any>>(
  config: FormConfig<T>,
): {
  values: T;
  setValue: <K extends keyof T>(field: K, value: T[K]) => void;
  validate: () => Record<keyof T, string | undefined>;
} {
  const values = {} as T;

  return {
    values,
    setValue(field, value) {
      values[field] = value;
    },
    validate() {
      const errors: Partial<Record<keyof T, string>> = {};

      for (const key in config) {
        const field = config[key];
        if (field.required && !values[key]) {
          errors[key] = `${field.label} is required`;
        }
        if (field.validate && values[key]) {
          errors[key] = field.validate(values[key]);
        }
      }

      return errors as Record<keyof T, string | undefined>;
    },
  };
}

// Usage
interface LoginForm {
  email: string;
  password: string;
}

const form = createForm<LoginForm>({
  email: {
    label: "Email",
    type: "email",
    required: true,
    validate: (value) => {
      return value.includes("@") ? undefined : "Invalid email";
    },
  },
  password: {
    label: "Password",
    type: "password",
    required: true,
    validate: (value) => {
      return value.length >= 8 ? undefined : "Password too short";
    },
  },
});

// ✅ Type-safe
form.setValue("email", "test@example.com");
form.setValue("password", "secret123");

// ❌ ERROR: Wrong field name
form.setValue("username", "test"); // ERROR: 'username' doesn't exist

// ❌ ERROR: Wrong type
form.setValue("email", 123); // ERROR: number not assignable to string
```

---

## 🎯 Part 2: Advanced Types

### Q4: What are conditional types? When would you use them?

**Answer:**

**Conditional types** select types based on conditions: `T extends U ? X : Y`

**Basic example:**

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false
```

**Real-world example 1: Extract promise type**

```typescript
type Unwrap<T> = T extends Promise<infer U> ? U : T;

type A = Unwrap<Promise<string>>; // string
type B = Unwrap<string>; // string
type C = Unwrap<Promise<Promise<number>>>; // Promise<number>

// Deep unwrap
type DeepUnwrap<T> = T extends Promise<infer U> ? DeepUnwrap<U> : T;

type D = DeepUnwrap<Promise<Promise<number>>>; // number
```

**Real-world example 2: Function return type extractor**

```typescript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function getUser() {
  return { id: 1, name: "Alice" };
}

type User = ReturnType<typeof getUser>; // { id: number; name: string }
```

**Real-world example 3: Conditional API response**

```typescript
type ApiResponse<T, E extends boolean = false> = E extends true
  ? { success: false; error: string }
  : { success: true; data: T };

function handleResponse<T, E extends boolean>(
  response: ApiResponse<T, E>,
): T | never {
  if (response.success) {
    return response.data; // ✅ TypeScript knows data exists
  } else {
    throw new Error(response.error); // ✅ TypeScript knows error exists
  }
}
```

**Real-world example 4: Type-safe prop extraction**

```typescript
// Extract all properties of a certain type
type PropertiesOfType<T, U> = {
  [K in keyof T]: T[K] extends U ? K : never;
}[keyof T];

interface User {
  id: number;
  name: string;
  age: number;
  email: string;
  isActive: boolean;
}

type StringProps = PropertiesOfType<User, string>; // 'name' | 'email'
type NumberProps = PropertiesOfType<User, number>; // 'id' | 'age'

// Use case: Type-safe sorting
function sortBy<T, K extends PropertiesOfType<T, string | number>>(
  items: T[],
  key: K,
): T[] {
  return items.sort((a, b) => {
    const aVal = a[key];
    const bVal = b[key];
    return aVal < bVal ? -1 : aVal > bVal ? 1 : 0;
  });
}

const users: User[] = [
  { id: 1, name: "Alice", age: 30, email: "alice@example.com", isActive: true },
];

sortBy(users, "name"); // ✅ OK
sortBy(users, "age"); // ✅ OK
sortBy(users, "isActive"); // ❌ ERROR: boolean not sortable
```

---

### Q5: Explain mapped types with examples

**Answer:**

**Mapped types** transform each property in a type.

**Basic syntax:**

```typescript
type Mapped<T> = {
  [K in keyof T]: /* transformation */;
};
```

**Example 1: Make all properties optional**

```typescript
type Partial<T> = {
  [K in keyof T]?: T[K];
};

interface User {
  id: number;
  name: string;
  email: string;
}

type PartialUser = Partial<User>;
// { id?: number; name?: string; email?: string; }
```

**Example 2: Make all properties readonly**

```typescript
type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};

const user: Readonly<User> = {
  id: 1,
  name: "Alice",
  email: "alice@example.com",
};

user.name = "Bob"; // ❌ ERROR: Cannot assign to 'name' (readonly)
```

**Example 3: Wrap all properties in Promise**

```typescript
type Promisify<T> = {
  [K in keyof T]: Promise<T[K]>;
};

interface User {
  id: number;
  name: string;
}

type AsyncUser = Promisify<User>;
// { id: Promise<number>; name: Promise<string>; }
```

**Real-world example 1: Type-safe getters**

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface State {
  count: number;
  user: { name: string };
}

type StateGetters = Getters<State>;
// {
//   getCount: () => number;
//   getUser: () => { name: string };
// }

// Implementation
class Store implements StateGetters {
  private state: State = { count: 0, user: { name: "Alice" } };

  getCount() {
    return this.state.count;
  }

  getUser() {
    return this.state.user;
  }
}
```

**Real-world example 2: Form field types**

```typescript
type FormField<T> = {
  value: T;
  error?: string;
  touched: boolean;
};

type FormFields<T> = {
  [K in keyof T]: FormField<T[K]>;
};

interface LoginData {
  email: string;
  password: string;
}

type LoginForm = FormFields<LoginData>;
// {
//   email: FormField<string>;
//   password: FormField<string>;
// }

const form: LoginForm = {
  email: {
    value: "",
    error: undefined,
    touched: false,
  },
  password: {
    value: "",
    error: undefined,
    touched: false,
  },
};
```

**Real-world example 3: API endpoint types**

```typescript
type ApiEndpoints = {
  "/users": { method: "GET"; response: User[] };
  "/users/:id": { method: "GET"; response: User };
  "/users": { method: "POST"; body: CreateUserDto; response: User };
};

type ExtractEndpoint<T, M extends string> = {
  [K in keyof T]: T[K] extends { method: M } ? K : never;
}[keyof T];

type GetEndpoints = ExtractEndpoint<ApiEndpoints, "GET">;
// '/users' | '/users/:id'

type PostEndpoints = ExtractEndpoint<ApiEndpoints, "POST">;
// '/users'
```

---

## 🎯 Part 3: Practical Application

### Q6: How do you type React components with TypeScript?

**Answer:**

**Functional components:**

```typescript
// ✅ Option 1: React.FC (includes children)
const Button: React.FC<{ onClick: () => void }> = ({ onClick, children }) => {
  return <button onClick={onClick}>{children}</button>;
};

// ✅ Option 2: Direct typing (more explicit)
interface ButtonProps {
  onClick: () => void;
  children: React.ReactNode;
}

const Button = ({ onClick, children }: ButtonProps) => {
  return <button onClick={onClick}>{children}</button>;
};

// ✅ Option 3: Generic component
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
}

function List<T>({ items, renderItem }: ListProps<T>) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage
<List
  items={[1, 2, 3]}
  renderItem={(num) => <span>{num}</span>}
/>
```

**Event handlers:**

```typescript
interface FormProps {
  onSubmit: (data: FormData) => void;
}

const Form = ({ onSubmit }: FormProps) => {
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    console.log(e.target.value);
  };

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    // ...
  };

  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    console.log('Clicked');
  };

  return (
    <form onSubmit={handleSubmit}>
      <input onChange={handleChange} />
      <button onClick={handleClick}>Submit</button>
    </form>
  );
};
```

**Hooks typing:**

```typescript
// useState
const [count, setCount] = useState<number>(0);
const [user, setUser] = useState<User | null>(null);

// useRef
const inputRef = useRef<HTMLInputElement>(null);
const timerRef = useRef<number | null>(null);

// useReducer
type State = { count: number };
type Action = { type: "increment" } | { type: "decrement" };

const reducer = (state: State, action: Action): State => {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    case "decrement":
      return { count: state.count - 1 };
  }
};

const [state, dispatch] = useReducer(reducer, { count: 0 });

// Custom hook
function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initialValue;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue] as const;
}

// Usage
const [theme, setTheme] = useLocalStorage<"light" | "dark">("theme", "light");
```

---

### Q7: How do you handle errors with TypeScript?

**Answer:**

**Type-safe error handling:**

```typescript
// ✅ Discriminated union for results
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) {
    return { success: false, error: "Division by zero" };
  }
  return { success: true, data: a / b };
}

// Usage (type-safe)
const result = divide(10, 2);

if (result.success) {
  console.log(result.data); // ✅ TypeScript knows data exists
} else {
  console.error(result.error); // ✅ TypeScript knows error exists
}
```

**Custom error classes:**

```typescript
class ValidationError extends Error {
  constructor(
    message: string,
    public field: string,
    public code: string,
  ) {
    super(message);
    this.name = "ValidationError";
  }
}

class NetworkError extends Error {
  constructor(
    message: string,
    public statusCode: number,
  ) {
    super(message);
    this.name = "NetworkError";
  }
}

// Type guard
function isValidationError(error: unknown): error is ValidationError {
  return error instanceof ValidationError;
}

function isNetworkError(error: unknown): error is NetworkError {
  return error instanceof NetworkError;
}

// Usage
try {
  await submitForm(data);
} catch (error) {
  if (isValidationError(error)) {
    console.error(`Validation error in ${error.field}: ${error.message}`);
  } else if (isNetworkError(error)) {
    console.error(`Network error (${error.statusCode}): ${error.message}`);
  } else {
    console.error("Unknown error:", error);
  }
}
```

**Async error handling:**

```typescript
type AsyncResult<T, E = Error> = Promise<Result<T, E>>;

async function fetchUser(id: string): AsyncResult<User, string> {
  try {
    const response = await fetch(`/api/users/${id}`);

    if (!response.ok) {
      return {
        success: false,
        error: `HTTP ${response.status}: ${response.statusText}`,
      };
    }

    const data = await response.json();
    return { success: true, data };
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error.message : "Unknown error",
    };
  }
}

// Usage
const result = await fetchUser("123");

if (result.success) {
  console.log(result.data.name);
} else {
  console.error(result.error);
}
```

---

## 🎯 Summary

**Core concepts:**

- ✅ `type` vs `interface` tradeoffs
- ✅ Structural typing (duck typing)
- ✅ Generics for reusable type-safe code
- ✅ Conditional types for type transformations
- ✅ Mapped types for property transformations

**Practical patterns:**

- ✅ Type-safe API wrappers
- ✅ Generic React components
- ✅ Discriminated unions for state
- ✅ Branded types for safety
- ✅ Type-safe error handling

**Interview tips:**

- Explain with real-world examples
- Show production tradeoffs
- Discuss type safety benefits
- Mention performance implications

Master these patterns for senior roles! 💪
