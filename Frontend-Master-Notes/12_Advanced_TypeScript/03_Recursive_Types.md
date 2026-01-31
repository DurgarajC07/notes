# Recursive Types in TypeScript

## Core Concept

Recursive types reference themselves in their definition, enabling representation of deeply nested or self-similar data structures like trees, JSON, and nested objects.

---

## Basic Recursive Type

```typescript
interface TreeNode {
  value: number;
  left?: TreeNode;
  right?: TreeNode;
}

const tree: TreeNode = {
  value: 1,
  left: {
    value: 2,
    left: { value: 4 },
    right: { value: 5 },
  },
  right: {
    value: 3,
  },
};
```

---

## Recursive Array Type

```typescript
type NestedArray<T> = T | NestedArray<T>[];

const nested: NestedArray<number> = [1, [2, [3, [4, 5]]], 6];
// Can be infinitely nested
```

---

## Deep Partial

Make all properties optional, recursively:

```typescript
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

interface User {
  name: string;
  address: {
    street: string;
    city: string;
    country: {
      name: string;
      code: string;
    };
  };
}

const partialUser: DeepPartial<User> = {
  name: "Alice",
  address: {
    city: "NYC",
    // street and country are optional
  },
};
```

---

## Deep Readonly

Make all properties readonly, recursively:

```typescript
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

interface Config {
  database: {
    host: string;
    port: number;
    credentials: {
      username: string;
      password: string;
    };
  };
}

const config: DeepReadonly<Config> = {
  database: {
    host: "localhost",
    port: 5432,
    credentials: {
      username: "admin",
      password: "secret",
    },
  },
};

// config.database.host = "newhost"; // ❌ Error: readonly
// config.database.credentials.password = "new"; // ❌ Error: readonly
```

---

## Deep Required

Make all properties required, recursively:

```typescript
type DeepRequired<T> = {
  [P in keyof T]-?: T[P] extends object ? DeepRequired<T[P]> : T[P];
};

interface PartialConfig {
  server?: {
    host?: string;
    port?: number;
  };
}

const fullConfig: DeepRequired<PartialConfig> = {
  server: {
    host: "localhost", // Both required
    port: 3000,
  },
};
```

---

## Path String Type

Generate all possible paths in an object:

```typescript
type PathImpl<T, K extends keyof T> = K extends string
  ? T[K] extends Record<string, any>
    ? T[K] extends ArrayLike<any>
      ? K | `${K}.${PathImpl<T[K], Exclude<keyof T[K], keyof any[]>>}`
      : K | `${K}.${PathImpl<T[K], keyof T[K]>}`
    : K
  : never;

type Path<T> = PathImpl<T, keyof T> | keyof T;

interface Data {
  user: {
    name: string;
    address: {
      city: string;
    };
  };
  posts: Array<{
    title: string;
  }>;
}

type DataPath = Path<Data>;
// Type: "user" | "posts" | "user.name" | "user.address" | "user.address.city"

function get<T, P extends Path<T>>(obj: T, path: P): any {
  // Implementation
}

const data: Data = {
  /* ... */
};
get(data, "user.address.city"); // ✅ Valid
get(data, "user.invalid"); // ❌ Error
```

---

## JSON Type

Type-safe JSON:

```typescript
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

interface JSONObject {
  [key: string]: JSONValue;
}

interface JSONArray extends Array<JSONValue> {}

const validJSON: JSONValue = {
  name: "Alice",
  age: 30,
  hobbies: ["reading", "coding"],
  address: {
    city: "NYC",
    zip: null,
  },
};
```

---

## Flatten Type

Flatten nested union types:

```typescript
type Flatten<T> = T extends Array<infer U> ? Flatten<U> : T;

type NestedNumbers = number[][][];
type FlatNumbers = Flatten<NestedNumbers>; // number

type Mixed = (string | number[])[];
type FlatMixed = Flatten<Mixed>; // string | number
```

---

## Deep Merge

Type-safe deep object merging:

```typescript
type DeepMerge<T, U> = {
  [K in keyof T | keyof U]: K extends keyof U
    ? K extends keyof T
      ? T[K] extends object
        ? U[K] extends object
          ? DeepMerge<T[K], U[K]>
          : U[K]
        : U[K]
      : U[K]
    : K extends keyof T
      ? T[K]
      : never;
};

interface Base {
  name: string;
  settings: {
    theme: string;
    notifications: boolean;
  };
}

interface Override {
  settings: {
    theme: string;
    language: string;
  };
  email: string;
}

type Merged = DeepMerge<Base, Override>;
// Type: {
//   name: string;
//   settings: {
//     theme: string;
//     notifications: boolean;
//     language: string;
//   };
//   email: string;
// }
```

---

## Deep Omit

Omit properties at any depth:

```typescript
type DeepOmit<T, K extends string> = T extends object
  ? {
      [P in keyof T as P extends K ? never : P]: P extends K
        ? never
        : DeepOmit<T[P], K>;
    }
  : T;

interface UserData {
  id: string;
  name: string;
  password: string;
  profile: {
    bio: string;
    password: string; // Also has password!
  };
}

type SafeUserData = DeepOmit<UserData, "password">;
// Type: {
//   id: string;
//   name: string;
//   profile: {
//     bio: string;
//   };
// }
```

---

## Deep Pick

Pick properties at any depth:

```typescript
type DeepPick<T, K extends string> = T extends object
  ? {
      [P in keyof T as P extends K ? P : never]: P extends K
        ? T[P]
        : DeepPick<T[P], K>;
    }
  : T;

interface Application {
  user: {
    id: string;
    name: string;
    email: string;
  };
  settings: {
    theme: string;
    id: string;
  };
}

type IdsOnly = DeepPick<Application, "id">;
// Type: {
//   user: { id: string };
//   settings: { id: string };
// }
```

---

## Nested Object Path Access

Type-safe nested object access:

```typescript
type PathValue<T, P extends string> = P extends `${infer K}.${infer Rest}`
  ? K extends keyof T
    ? PathValue<T[K], Rest>
    : never
  : P extends keyof T
    ? T[P]
    : never;

interface Config {
  server: {
    port: number;
    host: string;
    database: {
      name: string;
      port: number;
    };
  };
}

type ServerPort = PathValue<Config, "server.port">; // number
type DbName = PathValue<Config, "server.database.name">; // string
type Invalid = PathValue<Config, "server.invalid">; // never

// Runtime implementation
function getPath<T, P extends string>(obj: T, path: P): PathValue<T, P> {
  const keys = path.split(".");
  let result: any = obj;

  for (const key of keys) {
    result = result[key];
  }

  return result;
}

const config: Config = {
  server: {
    port: 3000,
    host: "localhost",
    database: {
      name: "mydb",
      port: 5432,
    },
  },
};

const port = getPath(config, "server.port"); // Type: number
const dbName = getPath(config, "server.database.name"); // Type: string
```

---

## Array Manipulation

### **Recursive Array Flattening**

```typescript
type DeepFlatten<T extends any[]> = T extends (infer U)[]
  ? U extends any[]
    ? DeepFlatten<U>
    : U
  : never;

type Nested = [1, [2, [3, [4, 5]]]];
type Flat = DeepFlatten<Nested>; // 1 | 2 | 3 | 4 | 5

// With depth limit
type FlattenDepth<T extends any[], Depth extends number = 1> = Depth extends 0
  ? T
  : T extends (infer U)[]
    ? U extends any[]
      ? FlattenDepth<U, [-1, 0, 1, 2, 3, 4, 5][Depth]>
      : U
    : T;

type DeepArray = [[[[number]]]];
type Flat1 = FlattenDepth<DeepArray, 1>; // [[number]][]
type Flat2 = FlattenDepth<DeepArray, 2>; // [number][]
```

---

## Recursive Union Types

### **Deeply Nested Unions**

```typescript
type NestedString = string | NestedString[];

const nested1: NestedString = "hello";
const nested2: NestedString = ["hello", ["world"]];
const nested3: NestedString = ["a", ["b", ["c", ["d"]]]];

// Extract all string values
type ExtractStrings<T> = T extends string
  ? T
  : T extends (infer U)[]
    ? ExtractStrings<U>
    : never;

type Strings = ExtractStrings<NestedString>; // string
```

---

## Form State Management

```typescript
type FormField<T> = {
  value: T;
  error?: string;
  touched: boolean;
};

type DeepFormState<T> = {
  [K in keyof T]: T[K] extends object ? DeepFormState<T[K]> : FormField<T[K]>;
};

interface UserForm {
  name: string;
  email: string;
  address: {
    street: string;
    city: string;
    zip: number;
  };
}

type FormState = DeepFormState<UserForm>;
// Type: {
//   name: FormField<string>;
//   email: FormField<string>;
//   address: {
//     street: FormField<string>;
//     city: FormField<string>;
//     zip: FormField<number>;
//   };
// }

// Usage
const formState: FormState = {
  name: { value: "John", touched: false },
  email: { value: "john@example.com", touched: true, error: "Invalid" },
  address: {
    street: { value: "123 Main St", touched: false },
    city: { value: "NYC", touched: false },
    zip: { value: 10001, touched: false },
  },
};
```

---

## Tree Operations

### **Binary Tree Type**

```typescript
interface BinaryTree<T> {
  value: T;
  left?: BinaryTree<T>;
  right?: BinaryTree<T>;
}

// Count nodes in tree
type CountNodes<T> = T extends BinaryTree<any>
  ? 1 + (T['left'] extends BinaryTree<any> ? CountNodes<T['left']> : 0) +
        (T['right'] extends BinaryTree<any> ? CountNodes<T['right']> : 0)
  : 0;

// Tree traversal types
type InOrder<T extends BinaryTree<any>> =
  T extends { left: infer L, value: infer V, right: infer R }
    ? [...(L extends BinaryTree<any> ? InOrder<L> : []), V, ...(R extends BinaryTree<any> ? InOrder<R> : [])]
    : [T['value']];

const tree = {
  value: 2,
  left: { value: 1 },
  right: { value: 3 }
} as const;

type Values = InOrder<typeof tree>; // [1, 2, 3]
```

---

## State Machine Types

```typescript
type State = {
  status: "idle" | "loading" | "success" | "error";
  data?: any;
  error?: string;
};

type Transition<S extends State> = {
  [K in S["status"]]: {
    [Action: string]: State;
  };
};

// Recursive state machine
type StateMachine<S extends State, T extends Transition<S>> = {
  state: S;
  dispatch: <A extends keyof T[S["status"]]>(
    action: A,
  ) => StateMachine<T[S["status"]][A], T>;
};

// Example usage
type AppState =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: string }
  | { status: "error"; error: string };

type AppTransitions = {
  idle: { FETCH: { status: "loading" } };
  loading: {
    SUCCESS: { status: "success"; data: string };
    ERROR: { status: "error"; error: string };
  };
  success: { REFETCH: { status: "loading" } };
  error: { RETRY: { status: "loading" } };
};
```

---

## Deep Freeze Type

```typescript
type DeepFreeze<T> = {
  readonly [P in keyof T]: T[P] extends Function
    ? T[P]
    : T[P] extends object
      ? DeepFreeze<T[P]>
      : T[P];
};

interface MutableConfig {
  api: {
    baseUrl: string;
    timeout: number;
    retry: {
      attempts: number;
      delay: number;
    };
  };
  features: {
    darkMode: boolean;
  };
}

type FrozenConfig = DeepFreeze<MutableConfig>;

const config: FrozenConfig = {
  api: {
    baseUrl: "https://api.example.com",
    timeout: 5000,
    retry: {
      attempts: 3,
      delay: 1000,
    },
  },
  features: {
    darkMode: true,
  },
};

// config.api.baseUrl = 'new'; // ❌ Error: readonly
// config.api.retry.attempts = 5; // ❌ Error: readonly
```

---

## Recursive Type Constraints

### **Maximum Depth Limiter**

```typescript
type RecursionLimit = [never, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

type DeepPartialWithLimit<
  T,
  Depth extends number = 5,
> = Depth extends RecursionLimit[0]
  ? T
  : {
      [P in keyof T]?: T[P] extends object
        ? DeepPartialWithLimit<T[P], RecursionLimit[Depth]>
        : T[P];
    };

// Prevents stack overflow on deeply nested types
interface VeryDeep {
  level1: {
    level2: {
      level3: {
        level4: {
          level5: {
            level6: string;
          };
        };
      };
    };
  };
}

type Limited = DeepPartialWithLimit<VeryDeep, 3>;
// Only goes 3 levels deep
```

---

## Validation Schema Types

```typescript
type ValidationRule = {
  type: "string" | "number" | "boolean" | "object" | "array";
  required?: boolean;
  min?: number;
  max?: number;
  pattern?: string;
};

type RecursiveSchema<T> = {
  [K in keyof T]: T[K] extends object
    ? T[K] extends any[]
      ? ValidationRule & { items: RecursiveSchema<T[K][number]> }
      : RecursiveSchema<T[K]>
    : ValidationRule;
};

interface User {
  name: string;
  age: number;
  address: {
    street: string;
    zip: number;
  };
  tags: string[];
}

const schema: RecursiveSchema<User> = {
  name: { type: "string", required: true, min: 2 },
  age: { type: "number", required: true, min: 0, max: 150 },
  address: {
    street: { type: "string", required: true },
    zip: { type: "number", required: true, min: 10000, max: 99999 },
  },
  tags: {
    type: "array",
    required: false,
    items: { type: "string" },
  },
};
```

---

## Cyclic Reference Handling

```typescript
interface Node {
  id: string;
  parent?: Node;
  children?: Node[];
}

// Safe cycle-aware operations
type NodeId<T extends { id: any }> = T["id"];

type ParentChain<T extends Node, Visited = never> = T["parent"] extends Node
  ? T["parent"]["id"] extends Visited
    ? [] // Cycle detected, stop
    : [
        T["parent"]["id"],
        ...ParentChain<T["parent"], Visited | T["parent"]["id"]>,
      ]
  : [];

// Usage prevents infinite recursion
const root: Node = { id: "root" };
const child: Node = { id: "child", parent: root };
root.children = [child];
child.parent = root; // Cyclic reference

type Chain = ParentChain<typeof child>; // ['root']
```

---

## Real-World Example: API Response Parser

```typescript
type APIResponse = {
  data: {
    user: {
      id: string;
      profile: {
        name: string;
        avatar: {
          url: string;
          thumbnail: {
            url: string;
          };
        };
      };
    };
    posts: Array<{
      id: string;
      author: {
        name: string;
      };
      comments: Array<{
        text: string;
        author: {
          name: string;
        };
      }>;
    }>;
  };
};

// Extract all URLs from nested structure
type ExtractUrls<T> = T extends { url: string }
  ? T["url"]
  : T extends object
    ? { [K in keyof T]: ExtractUrls<T[K]> }[keyof T]
    : never;

type AllUrls = ExtractUrls<APIResponse>; // string

// Extract all author names
type ExtractAuthors<T> = T extends { author: { name: string } }
  ? T["author"]["name"]
  : T extends object
    ? { [K in keyof T]: ExtractAuthors<T[K]> }[keyof T]
    : never;

type AllAuthors = ExtractAuthors<APIResponse>; // string
```

---

## Performance Considerations

### **⚠️ Recursion Depth Limits**

```typescript
// Bad - Can hit TypeScript's recursion limit (50 by default)
type DeepNesting = {
  level1: {
    level2: {
      level3: {
        // ... 50+ levels
      };
    };
  };
};

// Good - Use tail recursion or depth limits
type SafeDeepPartial<T, Depth extends number = 10> = Depth extends 0
  ? T
  : { [P in keyof T]?: SafeDeepPartial<T[P], Decrement<Depth>> };
```

### **✅ Optimization Tips**

1. **Limit recursion depth** with counter types
2. **Use conditional types** to exit early
3. **Cache intermediate results** with type aliases
4. **Avoid circular references** in type definitions
5. **Test with `tsc --noEmit`** to catch compilation issues

---

## Best Practices

### ✅ DO: Set Recursion Limits

```typescript
// Good - Explicit depth limit
type DeepReadonly<T, Depth extends number = 5> = Depth extends 0
  ? T
  : { readonly [P in keyof T]: DeepReadonly<T[P], Dec<Depth>> };
```

### ❌ DON'T: Unbounded Recursion

```typescript
// Bad - Can cause stack overflow in TypeScript compiler
type Infinite<T> = { [K in keyof T]: Infinite<T[K]> };
```

### ✅ DO: Handle Arrays and Primitives

```typescript
// Good - Checks for arrays and primitives
type DeepType<T> = T extends any[]
  ? DeepType<T[number]>[]
  : T extends object
    ? { [K in keyof T]: DeepType<T[K]> }
    : T;
```

### ✅ DO: Use Utility Types as Building Blocks

```typescript
// Good - Compose smaller utility types
type DeepNullable<T> = DeepPartial<DeepReadonly<T>>;
```

---

## Key Takeaways

1. **Recursive types reference themselves** enabling deeply nested structures
2. **DeepPartial/DeepReadonly/DeepRequired** transform all nested properties
3. **Path types** enable type-safe nested object access
4. **Recursion limits** prevent TypeScript compiler stack overflow
5. **Handle arrays and primitives** separately from objects
6. **Tree structures** naturally benefit from recursive types
7. **Form state management** uses recursive types for nested validation
8. **JSON types** are inherently recursive (arrays and objects)
9. **Use tail recursion** or depth counters for performance
10. **Real-world APIs** often have deeply nested responses requiring recursive types
    config: {
    theme: string;
    };
    }

interface Override {
config: {
mode: string;
};
}

type Merged = DeepMerge<Base, Override>;
// {
// name: string;
// config: {
// theme: string;
// mode: string;
// };
// }

````

---

## Recursion Depth Limit

TypeScript has a recursion depth limit (~50 levels):

```typescript
// ⚠️ This might hit recursion limit
type VeryDeep = DeepPartial</* 100 levels deep object */>;

// ✅ Solution: Add depth counter
type DeepPartialWithLimit<T, D extends number = 5> =
  D extends 0
    ? T
    : {
        [P in keyof T]?: T[P] extends object
          ? DeepPartialWithLimit<T[P], Prev<D>>
          : T[P];
      };

type Prev<N extends number> = [-1, 0, 1, 2, 3, 4, 5][N];
````

---

## Best Practices

✅ **Use recursive types** for tree structures and deep nesting  
✅ **Add depth limits** for very deep recursion  
✅ **Combine with conditional types** for flexibility  
✅ **Test with real data** - ensure performance is acceptable  
❌ **Don't recurse infinitely** - hit compiler limits  
❌ **Avoid overly complex** recursive types - hard to debug

---

## Key Takeaways

1. **Recursive types reference themselves** in definition
2. **Perfect for tree structures** and nested data
3. **Deep utilities** (DeepPartial, DeepReadonly) are powerful
4. **Path types enable type-safe access** to nested properties
5. **Watch recursion depth** - TypeScript has limits
6. **Combine with mapped types** for transformations
