# TypeScript Inference Optimization

## Core Concept

TypeScript's type inference is powerful but can be a performance bottleneck. Understanding when to use explicit types versus letting TypeScript infer them is crucial for large-scale applications.

---

## Type Inference Performance

### How TypeScript Infers Types

```typescript
// TypeScript must analyze entire function to infer return type
function processData(data: unknown[]) {
  const result = data.map((item) => {
    if (typeof item === "string") {
      return item.toUpperCase();
    }
    if (typeof item === "number") {
      return item * 2;
    }
    return null;
  });

  // Inferred type: (string | number | null)[]
  return result;
}

// Explicit return type - TypeScript can validate incrementally
function processDataExplicit(data: unknown[]): (string | number | null)[] {
  const result = data.map((item) => {
    if (typeof item === "string") {
      return item.toUpperCase();
    }
    if (typeof item === "number") {
      return item * 2;
    }
    return null;
  });

  return result;
}
```

### When Explicit Types Help Performance

```typescript
// ❌ Slow - TypeScript must infer complex return type
const complexOperation = (data: any[]) => {
  return data.reduce(
    (acc, item) => {
      if (item.type === "user") {
        acc.users.push(item);
      } else if (item.type === "product") {
        acc.products.push(item);
      } else {
        acc.others.push(item);
      }
      return acc;
    },
    { users: [], products: [], others: [] },
  );
};

// ✅ Fast - Explicit types enable incremental checking
interface GroupedData {
  users: User[];
  products: Product[];
  others: unknown[];
}

const complexOperationExplicit = (data: any[]): GroupedData => {
  return data.reduce<GroupedData>(
    (acc, item) => {
      if (item.type === "user") {
        acc.users.push(item);
      } else if (item.type === "product") {
        acc.products.push(item);
      } else {
        acc.others.push(item);
      }
      return acc;
    },
    { users: [], products: [], others: [] },
  );
};
```

---

## Function Parameter Inference

### Contextual Typing

```typescript
// ❌ Poor inference - TypeScript must work harder
const users = [
  { name: "Alice", age: 30 },
  { name: "Bob", age: 25 },
];

// TypeScript infers callback parameter types
users.map((user) => user.name); // Works but slower compilation

// ✅ Better - Explicit interface helps
interface User {
  name: string;
  age: number;
}

const typedUsers: User[] = [
  { name: "Alice", age: 30 },
  { name: "Bob", age: 25 },
];

// Faster - TypeScript knows callback parameter type immediately
typedUsers.map((user: User) => user.name);
```

### Generic Inference Optimization

```typescript
// ❌ Slow - Generic inference across multiple calls
function compose(f: any, g: any): any {
  return (x: any) => f(g(x));
}

const addOne = (x: number) => x + 1;
const double = (x: number) => x * 2;
const result = compose(double, addOne)(5);

// ✅ Fast - Explicit generics
function composeExplicit<A, B, C>(f: (b: B) => C, g: (a: A) => B): (a: A) => C {
  return (x: A) => f(g(x));
}

const resultExplicit = composeExplicit<number, number, number>(
  double,
  addOne,
)(5);

// ✅ Even better - Type parameters with constraints
function composeSafe<A, B extends A, C>(
  f: (b: B) => C,
  g: (a: A) => B,
): (a: A) => C {
  return (x: A) => f(g(x));
}
```

---

## Array and Object Literal Inference

### Array Type Assertions

```typescript
// ❌ Slow - TypeScript infers widest possible type
const mixedArray = [1, "two", 3, "four"];
// Inferred: (string | number)[]

// Must check each usage
const first = mixedArray[0]; // string | number
if (typeof first === "number") {
  console.log(first * 2);
}

// ✅ Fast - Explicit tuple or union array
const explicitArray: [number, string, number, string] = [1, "two", 3, "four"];
const firstExplicit = explicitArray[0]; // number - no check needed

// Or use const assertion for literal types
const literalArray = [1, "two", 3, "four"] as const;
const firstLiteral = literalArray[0]; // 1 (literal type)
```

### Object Property Inference

```typescript
// ❌ Slow - Deep object inference
const config = {
  api: {
    baseURL: "https://api.example.com",
    timeout: 5000,
    headers: {
      "Content-Type": "application/json",
      Authorization: "Bearer token",
    },
  },
  features: {
    darkMode: true,
    notifications: false,
  },
};

// TypeScript must infer nested structure on every access
const url = config.api.baseURL;

// ✅ Fast - Explicit interface
interface Config {
  api: {
    baseURL: string;
    timeout: number;
    headers: Record<string, string>;
  };
  features: {
    darkMode: boolean;
    notifications: boolean;
  };
}

const typedConfig: Config = {
  api: {
    baseURL: "https://api.example.com",
    timeout: 5000,
    headers: {
      "Content-Type": "application/json",
      Authorization: "Bearer token",
    },
  },
  features: {
    darkMode: true,
    notifications: false,
  },
};
```

---

## Conditional Type Inference

### Lazy Evaluation

```typescript
// ❌ Slow - Complex inference chain
type ExtractArrayType<T> = T extends (infer U)[] ? U : never;
type ExtractPromiseType<T> = T extends Promise<infer U> ? U : never;
type ExtractFunctionReturn<T> = T extends (...args: any[]) => infer R
  ? R
  : never;

type ChainedInference<T> = ExtractFunctionReturn<
  ExtractPromiseType<ExtractArrayType<T>>
>;

// TypeScript must evaluate all conditionals
type Result = ChainedInference<(() => Promise<string[]>)[]>;

// ✅ Fast - Single conditional with explicit type
type ChainedInferenceOptimized<T> = T extends (() => Promise<(infer U)[]>)[]
  ? U
  : never;

type ResultOptimized = ChainedInferenceOptimized<(() => Promise<string[]>)[]>;
```

### Helper Type Optimization

```typescript
// ❌ Slow - Recursive inference
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

interface NestedData {
  user: {
    profile: {
      name: string;
      settings: {
        theme: string;
        language: string;
      };
    };
  };
}

type ReadonlyNested = DeepReadonly<NestedData>;

// ✅ Fast - Explicit readonly at definition
interface ReadonlyNestedData {
  readonly user: {
    readonly profile: {
      readonly name: string;
      readonly settings: {
        readonly theme: string;
        readonly language: string;
      };
    };
  };
}
```

---

## Real-World Pattern: API Response Typing

### Before Optimization

```typescript
// ❌ Slow - TypeScript infers from usage
async function fetchUser(id: string) {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();

  // TypeScript must infer data type from usage
  return {
    id: data.id,
    name: data.name,
    email: data.email,
    profile: {
      bio: data.profile?.bio,
      avatar: data.profile?.avatar,
    },
  };
}

// Each call requires type inference
const user = await fetchUser("123");
console.log(user.name); // What type is this?
```

### After Optimization

```typescript
// ✅ Fast - Explicit types enable incremental checking
interface UserProfile {
  bio: string;
  avatar: string;
}

interface User {
  id: string;
  name: string;
  email: string;
  profile: UserProfile;
}

interface ApiResponse<T> {
  data: T;
  status: number;
  message?: string;
}

async function fetchUserOptimized(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data: ApiResponse<User> = await response.json();

  return {
    id: data.data.id,
    name: data.data.name,
    email: data.data.email,
    profile: {
      bio: data.data.profile.bio,
      avatar: data.data.profile.avatar,
    },
  };
}

// No inference needed - type is explicit
const userOptimized = await fetchUserOptimized("123");
console.log(userOptimized.name); // string
```

---

## Real-World Pattern: React Component Props

### Before Optimization

```typescript
// ❌ Slow - Props inferred from usage
function UserCard({ user, onEdit, onDelete }) {
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <button onClick={() => onEdit(user)}>Edit</button>
      <button onClick={() => onDelete(user.id)}>Delete</button>
    </div>
  );
}

// TypeScript must infer prop types from JSX usage
<UserCard
  user={{ name: 'Alice', email: 'alice@example.com' }}
  onEdit={(u) => console.log(u)}
  onDelete={(id) => console.log(id)}
/>
```

### After Optimization

```typescript
// ✅ Fast - Explicit prop interface
interface User {
  id: string;
  name: string;
  email: string;
}

interface UserCardProps {
  user: User;
  onEdit: (user: User) => void;
  onDelete: (id: string) => void;
}

function UserCardOptimized({ user, onEdit, onDelete }: UserCardProps) {
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <button onClick={() => onEdit(user)}>Edit</button>
      <button onClick={() => onDelete(user.id)}>Delete</button>
    </div>
  );
}

// No inference needed
<UserCardOptimized
  user={{ id: '1', name: 'Alice', email: 'alice@example.com' }}
  onEdit={(u: User) => console.log(u)}
  onDelete={(id: string) => console.log(id)}
/>
```

---

## Real-World Pattern: State Management

### Before Optimization

```typescript
// ❌ Slow - Action types inferred from reducer
function reducer(state, action) {
  switch (action.type) {
    case "SET_USER":
      return { ...state, user: action.payload };
    case "SET_LOADING":
      return { ...state, loading: action.payload };
    case "SET_ERROR":
      return { ...state, error: action.payload };
    default:
      return state;
  }
}

// TypeScript must infer action shape from usage
dispatch({ type: "SET_USER", payload: { name: "Alice" } });
```

### After Optimization

```typescript
// ✅ Fast - Explicit action and state types
interface User {
  name: string;
  email: string;
}

interface AppState {
  user: User | null;
  loading: boolean;
  error: string | null;
}

type Action =
  | { type: "SET_USER"; payload: User }
  | { type: "SET_LOADING"; payload: boolean }
  | { type: "SET_ERROR"; payload: string };

function reducerOptimized(state: AppState, action: Action): AppState {
  switch (action.type) {
    case "SET_USER":
      return { ...state, user: action.payload };
    case "SET_LOADING":
      return { ...state, loading: action.payload };
    case "SET_ERROR":
      return { ...state, error: action.payload };
  }
}

// Type-safe dispatch with no inference needed
dispatch({
  type: "SET_USER",
  payload: { name: "Alice", email: "alice@example.com" },
});
```

---

## Performance Benchmarks

### Compilation Time Comparison

```typescript
// Test file: 1000 function calls

// ❌ Inferred types: ~850ms compilation
const inferredFunctions = Array.from({ length: 1000 }, (_, i) => {
  return (data: any) => {
    return data.map((item: any) => item.id);
  };
});

// ✅ Explicit types: ~320ms compilation
const explicitFunctions = Array.from({ length: 1000 }, (_, i) => {
  return (data: Array<{ id: string }>): string[] => {
    return data.map((item) => item.id);
  };
});
```

### IDE Performance Impact

```typescript
// ❌ Slow autocomplete - TypeScript must infer on every keystroke
const processItem = (item: any) => {
  return {
    processed: true,
    data: item.data,
    timestamp: Date.now(),
  };
};

// ✅ Fast autocomplete - Types known immediately
interface Item {
  data: string;
}

interface ProcessedItem {
  processed: boolean;
  data: string;
  timestamp: number;
}

const processItemExplicit = (item: Item): ProcessedItem => {
  return {
    processed: true,
    data: item.data,
    timestamp: Date.now(),
  };
};
```

---

## Best Practices

### 1. Explicit Return Types for Public APIs

```typescript
// ✅ Always explicit for exported functions
export function fetchData(id: string): Promise<Data> {
  return api.get(`/data/${id}`);
}

// ✅ Always explicit for class methods
export class DataService {
  async fetchData(id: string): Promise<Data> {
    return api.get(`/data/${id}`);
  }
}
```

### 2. Type Parameters Over Inference

```typescript
// ❌ Avoid
const result = useState(null); // useState<null>

// ✅ Prefer
const result = useState<User | null>(null); // useState<User | null>
```

### 3. Const Assertions for Literals

```typescript
// ❌ Widened types
const config = {
  mode: "development",
  port: 3000,
};
// { mode: string, port: number }

// ✅ Literal types
const configLiteral = {
  mode: "development",
  port: 3000,
} as const;
// { mode: 'development', port: 3000 }
```

### 4. Annotate Complex Generics

```typescript
// ❌ Hard to infer
const data = useQuery(fetchUser);

// ✅ Explicit
const data = useQuery<User, Error>(fetchUser);
```

---

## Key Takeaways

1. **Explicit return types** improve compilation speed and IDE performance
2. **Type parameters** prevent expensive inference for generics
3. **Interface definitions** enable incremental type checking
4. **Const assertions** provide literal types without inference
5. **Explicit generics** in hooks and utilities reduce inference cost
6. Balance between **DRY principles** and **performance** - some redundancy is okay
7. Measure impact with `tsc --diagnostics` and `--extendedDiagnostics`
8. Profile with VS Code's **TypeScript: Restart TS Server** and check response times
9. Explicit types in **public APIs** benefit both performance and developer experience
10. Use **ESLint rules** to enforce explicit return types: `@typescript-eslint/explicit-function-return-type`
