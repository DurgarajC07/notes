# Type Complexity and Recursion Limits

## Core Concept

TypeScript has limits on type complexity to prevent infinite loops and performance issues. Understanding these limits and how to work within them ensures maintainable, performant type definitions.

---

## Recursion Depth Limit

```typescript
// TypeScript has a recursion depth limit (default ~50)

// ❌ Too deep - will error
type DeepNest<T, N extends number = 0> = N extends 50
  ? T
  : { value: DeepNest<T, [N, ...any[]]> };

// ✅ Limited recursion
type SafeDeepPartial<T, Depth extends number = 3> = Depth extends 0
  ? T
  : {
      [K in keyof T]?: T[K] extends object
        ? SafeDeepPartial<T[K], [-1, 0, 1, 2, 3][Depth]>
        : T[K];
    };
```

---

## Instantiation Depth

```typescript
// ❌ Causes "Type instantiation is excessively deep and possibly infinite"
type BadRecursive<T> = {
  [K in keyof T]: T[K] extends object
    ? BadRecursive<T[K]> // No termination condition!
    : T[K];
};

// ✅ Proper termination
type GoodRecursive<T, Depth extends number = 5> = Depth extends 0
  ? T
  : {
      [K in keyof T]: T[K] extends object
        ? GoodRecursive<T[K], Prev<Depth>>
        : T[K];
    };

type Prev<N extends number> = [-1, 0, 1, 2, 3, 4, 5][N] extends infer P extends
  number
  ? P
  : never;
```

---

## Union Distribution Limits

```typescript
// ❌ Too many unions cause performance issues
type BadUnion =
  'a' | 'b' | 'c' | 'd' | 'e' | /* ... 1000 more */ | 'zzz';

type Permutations<T extends string> =
  T extends any
    ? `${T}-${Permutations<Exclude<T, T>>}`
    : never;

// With 10 items, this creates 10! = 3,628,800 types!

// ✅ Limit union size
type SafeUnion = 'small' | 'medium' | 'large'; // Keep unions reasonable
```

---

## Conditional Type Complexity

```typescript
// ❌ Nested conditionals get exponentially complex
type Complex<T> = T extends string
  ? T extends `${infer A}${infer B}`
    ? A extends `${infer C}${infer D}`
      ? C extends `${infer E}${infer F}`
        ? [E, F, D, B] // Too deep!
        : never
      : never
    : never
  : never;

// ✅ Break into smaller types
type ParseFirst<T extends string> = T extends `${infer First}${infer Rest}`
  ? [First, Rest]
  : never;

type ParseSecond<T extends string> = T extends `${infer Second}${infer Rest}`
  ? [Second, Rest]
  : never;

type Better<T> = T extends string
  ? ParseFirst<T> extends [infer A, infer B extends string]
    ? ParseSecond<B> extends [infer C, infer D]
      ? [A, C, D]
      : never
    : never
  : never;
```

---

## Mapped Type Optimization

```typescript
// ❌ Slow - creates intermediate types
type Slow<T> = {
  [K in keyof T]: {
    [P in keyof T[K]]: {
      [Q in keyof T[K][P]]: T[K][P][Q];
    };
  };
};

// ✅ Faster - single pass
type Fast<T> = {
  [K in keyof T]: T[K] extends object ? { [P in keyof T[K]]: T[K][P] } : T[K];
};
```

---

## Defer Type Evaluation

```typescript
// ❌ Evaluates immediately
type Immediate<T> = T extends object ? { [K in keyof T]: T[K] } : T;

// ✅ Defers evaluation
type Deferred<T> = T extends object
  ? T extends infer O
    ? { [K in keyof O]: O[K] }
    : never
  : T;
```

---

## Type Caching

```typescript
// ❌ Recomputes every time
type NoCaching<T> = {
  field1: ComplexType<T>;
  field2: ComplexType<T>;
  field3: ComplexType<T>;
};

// ✅ Cache in intermediate type
type Cached<T> = ComplexType<T>;

type WithCaching<T> = {
  field1: Cached<T>;
  field2: Cached<T>;
  field3: Cached<T>;
};
```

---

## Limit Template Literals

```typescript
// ❌ Exponential expansion
type BadTemplate<A extends string, B extends string> = `${A}-${B}`; // If A has 100 options and B has 100, creates 10,000 types!

// ✅ Use constraints
type GoodTemplate<T extends "small" | "medium" | "large"> = `size-${T}`;
```

---

## Simplify Infer

```typescript
// ❌ Complex infer pattern
type Complex<T> = T extends (...args: infer A) => infer R
  ? A extends [infer First, ...infer Rest]
    ? Rest extends [infer Second, ...infer Others]
      ? [First, Second, R]
      : never
    : never
  : never;

// ✅ Simplified
type Parameters<T> = T extends (...args: infer P) => any ? P : never;
type ReturnType<T> = T extends (...args: any) => infer R ? R : never;

type Simple<T> = T extends (...args: any) => any
  ? [Parameters<T>[0], Parameters<T>[1], ReturnType<T>]
  : never;
```

---

## Avoid Circular References

```typescript
// ❌ Circular type
interface Node {
  value: number;
  children: Node[]; // Direct self-reference - can cause issues
}

// ✅ Use interface merging or conditional
interface NodeBase {
  value: number;
}

interface Node extends NodeBase {
  children?: Node[];
}
```

---

## Measure Type Complexity

```typescript
// Use @typescript-eslint/ban-types to catch overly complex types

// tsconfig.json
{
  "compilerOptions": {
    "noImplicitAny": true
  }
}

// Check compilation time
// tsc --noEmit --diagnostics
```

---

## Real-World: Deep Partial

```typescript
// ❌ Unbounded recursion
type BadDeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? BadDeepPartial<T[K]> : T[K];
};

// ✅ Bounded with depth limit
type DeepPartial<T, MaxDepth extends number = 10> = MaxDepth extends 0
  ? T
  : {
      [K in keyof T]?: T[K] extends object
        ? DeepPartial<T[K], Decrement<MaxDepth>>
        : T[K];
    };

type Decrement<N extends number> = [-1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10][N];

// Usage with safe depth
interface User {
  name: string;
  address: {
    street: string;
    city: {
      name: string;
      country: string;
    };
  };
}

type PartialUser = DeepPartial<User, 5>; // Safely limited
```

---

## Configuration Limits

```json
// tsconfig.json - increase limits (use carefully)
{
  "compilerOptions": {
    // No direct config for recursion limit
    // But you can disable checks (not recommended)
    "noEmit": true,
    "skipLibCheck": true
  }
}
```

---

## Best Practices

✅ **Limit recursion depth** to 5-10 levels  
✅ **Break complex types** into smaller ones  
✅ **Cache intermediate types**  
✅ **Avoid large unions** (keep < 100 members)  
✅ **Use constraints** on template literals  
✅ **Test type performance** with `--diagnostics`  
❌ **Don't create unbounded recursion**  
❌ **Don't generate massive union permutations**  
❌ **Don't nest conditionals deeply**

---

## Key Takeaways

1. **TypeScript has recursion depth limits** (~50 levels)
2. **Instantiation depth errors** mean infinite loops
3. **Break complex types** into smaller pieces
4. **Cache computed types** to avoid recalculation
5. **Limit union sizes** to prevent exponential expansion
6. **Use `--diagnostics`** to measure complexity
7. **Bounded recursion** is safer than unbounded
