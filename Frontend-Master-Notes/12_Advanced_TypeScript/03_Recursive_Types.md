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
//   name: string;
//   config: {
//     theme: string;
//     mode: string;
//   };
// }
```

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
```

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
