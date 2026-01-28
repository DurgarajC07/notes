# Type-Level Programming

## Core Concept

Type-level programming uses TypeScript's type system to perform computations at compile time, creating powerful generic utilities and enforcing complex constraints.

---

## Conditional Types

```typescript
// Basic conditional type
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false

// Extract function return type
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function getUser() {
  return { id: 1, name: "John" };
}

type User = ReturnType<typeof getUser>; // { id: number; name: string }

// Extract promise value
type Awaited<T> = T extends Promise<infer U> ? U : T;

type A = Awaited<Promise<string>>; // string
type B = Awaited<number>; // number
```

---

## Mapped Types

```typescript
// Make all properties optional
type Partial<T> = {
  [P in keyof T]?: T[P];
};

// Make all properties required
type Required<T> = {
  [P in keyof T]-?: T[P]; // -? removes optional
};

// Make all properties readonly
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};

// Pick specific properties
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Omit specific properties
type Omit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;

// Example
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}

type UserUpdate = Partial<User>; // All optional
type UserCreate = Required<Omit<User, "id">>; // All required except id
type PublicUser = Omit<User, "password">; // No password
```

---

## Template Literal Types

```typescript
// Uppercase/Lowercase/Capitalize/Uncapitalize
type Greeting = "hello world";
type LoudGreeting = Uppercase<Greeting>; // "HELLO WORLD"
type QuietGreeting = Lowercase<Greeting>; // "hello world"
type Capitalized = Capitalize<Greeting>; // "Hello world"

// Generate event names
type EventName<T extends string> = `on${Capitalize<T>}`;

type ClickEvent = EventName<"click">; // "onClick"
type HoverEvent = EventName<"hover">; // "onHover"

// Generate getter/setter names
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type Setters<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

interface State {
  name: string;
  age: number;
}

type StateGetters = Getters<State>;
// {
//   getName: () => string;
//   getAge: () => number;
// }

type StateSetters = Setters<State>;
// {
//   setName: (value: string) => void;
//   setAge: (value: number) => void;
// }
```

---

## Recursive Types

```typescript
// JSON type
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

const data: JSONValue = {
  name: "John",
  age: 30,
  hobbies: ["coding", "gaming"],
  address: {
    city: "NYC",
    zip: 10001,
  },
};

// Nested path type
type PathImpl<T, Key extends keyof T> = Key extends string
  ? T[Key] extends Record<string, any>
    ? `${Key}.${Path<T[Key]>}` | Key
    : Key
  : never;

type Path<T> = PathImpl<T, keyof T> | keyof T;

interface Nested {
  user: {
    profile: {
      name: string;
    };
    settings: {
      theme: string;
    };
  };
}

type NestedPaths = Path<Nested>;
// "user" | "user.profile" | "user.profile.name" | "user.settings" | "user.settings.theme"
```

---

## Utility Type: DeepPartial

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
}

const partialConfig: DeepPartial<Config> = {
  server: {
    port: 3000, // Other fields optional
  },
};
```

---

## Utility Type: DeepReadonly

```typescript
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

interface MutableState {
  user: {
    name: string;
    settings: {
      theme: string;
    };
  };
}

type ImmutableState = DeepReadonly<MutableState>;

const state: ImmutableState = {
  user: {
    name: "John",
    settings: { theme: "dark" },
  },
};

state.user.name = "Jane"; // ❌ Error: readonly
state.user.settings.theme = "light"; // ❌ Error: readonly
```

---

## Tuple Types

```typescript
// Get first element type
type First<T extends any[]> = T extends [infer F, ...any[]] ? F : never;

type A = First<[string, number, boolean]>; // string

// Get last element type
type Last<T extends any[]> = T extends [...any[], infer L] ? L : never;

type B = Last<[string, number, boolean]>; // boolean

// Tuple to union
type TupleToUnion<T extends any[]> = T[number];

type C = TupleToUnion<[string, number, boolean]>; // string | number | boolean

// Prepend to tuple
type Prepend<T extends any[], E> = [E, ...T];

type D = Prepend<[2, 3], 1>; // [1, 2, 3]
```

---

## Union Manipulation

```typescript
// Exclude from union
type Exclude<T, U> = T extends U ? never : T;

type A = Exclude<"a" | "b" | "c", "a">; // 'b' | 'c'

// Extract from union
type Extract<T, U> = T extends U ? T : never;

type B = Extract<"a" | "b" | "c", "a" | "b">; // 'a' | 'b'

// Non-nullable
type NonNullable<T> = T extends null | undefined ? never : T;

type C = NonNullable<string | null | undefined>; // string

// Union to intersection
type UnionToIntersection<U> = (U extends any ? (k: U) => void : never) extends (
  k: infer I,
) => void
  ? I
  : never;

type D = UnionToIntersection<{ a: string } | { b: number }>;
// { a: string } & { b: number }
```

---

## Function Utilities

```typescript
// Get function parameters
type Parameters<T extends (...args: any) => any> = T extends (
  ...args: infer P
) => any
  ? P
  : never;

function greet(name: string, age: number) {
  return `Hello ${name}, you are ${age}`;
}

type GreetParams = Parameters<typeof greet>; // [string, number]

// Constructor parameters
type ConstructorParameters<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: infer P) => any ? P : never;

class Person {
  constructor(name: string, age: number) {}
}

type PersonParams = ConstructorParameters<typeof Person>; // [string, number]

// Instance type
type InstanceType<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: any) => infer R ? R : any;

type PersonInstance = InstanceType<typeof Person>; // Person
```

---

## Branded Type Factory

```typescript
declare const brand: unique symbol;

type Brand<T, TBrand> = T & { [brand]: TBrand };

type BrandedFactory<T, TBrand extends string> = {
  create: (value: T) => Brand<T, TBrand>;
  unwrap: (branded: Brand<T, TBrand>) => T;
};

function createBrandedFactory<T, TBrand extends string>(
  validate?: (value: T) => boolean,
): BrandedFactory<T, TBrand> {
  return {
    create: (value: T) => {
      if (validate && !validate(value)) {
        throw new Error("Validation failed");
      }
      return value as Brand<T, TBrand>;
    },
    unwrap: (branded: Brand<T, TBrand>) => branded as T,
  };
}

// Usage
const Email = createBrandedFactory<string, "Email">((s) =>
  /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(s),
);

const email = Email.create("user@example.com");
const rawEmail = Email.unwrap(email);
```

---

## Builder Pattern with Types

```typescript
type Builder<T> = {
  [K in keyof T]-?: (value: T[K]) => Builder<T>;
} & {
  build: () => T;
};

function createBuilder<T>(): Builder<T> {
  const values: Partial<T> = {};

  const builder: any = {
    build: () => values as T,
  };

  return new Proxy(builder, {
    get(target, prop: string) {
      if (prop === "build") {
        return target.build;
      }
      return (value: any) => {
        values[prop as keyof T] = value;
        return builder;
      };
    },
  });
}

interface User {
  id: number;
  name: string;
  email: string;
}

const user = createBuilder<User>()
  .id(1)
  .name("John")
  .email("john@example.com")
  .build();
```

---

## Type-Safe Event Emitter

```typescript
type EventMap = Record<string, any>;

type EventKey<T extends EventMap> = string & keyof T;
type EventReceiver<T> = (params: T) => void;

class TypedEventEmitter<T extends EventMap> {
  private listeners: {
    [K in keyof T]?: Array<EventReceiver<T[K]>>;
  } = {};

  on<K extends EventKey<T>>(eventName: K, handler: EventReceiver<T[K]>): void {
    if (!this.listeners[eventName]) {
      this.listeners[eventName] = [];
    }
    this.listeners[eventName]!.push(handler);
  }

  emit<K extends EventKey<T>>(eventName: K, params: T[K]): void {
    const handlers = this.listeners[eventName];
    if (handlers) {
      handlers.forEach((handler) => handler(params));
    }
  }
}

// Usage
interface Events {
  login: { userId: string; timestamp: Date };
  logout: { userId: string };
  error: { message: string; code: number };
}

const emitter = new TypedEventEmitter<Events>();

emitter.on("login", (data) => {
  // data is typed as { userId: string; timestamp: Date }
  console.log(data.userId, data.timestamp);
});

emitter.emit("login", {
  userId: "123",
  timestamp: new Date(),
});

emitter.emit("login", { userId: "123" }); // ❌ Error: missing timestamp
```

---

## Matrix Type Operations

```typescript
// Matrix multiplication at type level
type Length<T extends any[]> = T["length"];

type BuildTuple<L extends number, T extends any[] = []> = T["length"] extends L
  ? T
  : BuildTuple<L, [any, ...T]>;

type Add<A extends number, B extends number> = Length<
  [...BuildTuple<A>, ...BuildTuple<B>]
>;

type Subtract<A extends number, B extends number> =
  BuildTuple<A> extends [...BuildTuple<B>, ...infer R] ? Length<R> : never;

type Multiply<
  A extends number,
  B extends number,
  Acc extends any[] = [],
> = B extends 0
  ? Length<Acc>
  : Multiply<A, Subtract<B, 1>, [...Acc, ...BuildTuple<A>]>;

type Result1 = Add<3, 5>; // 8
type Result2 = Subtract<10, 3>; // 7
type Result3 = Multiply<3, 4>; // 12
```

---

## Best Practices

✅ **Use built-in utilities** before creating custom ones  
✅ **Keep type logic simple** - complex types hurt readability  
✅ **Document complex types** with examples  
✅ **Test types** with type assertions  
✅ **Leverage inference** with `infer` keyword  
❌ **Don't over-engineer** - simple is better  
❌ **Don't abuse type system** - it's not a runtime language  
❌ **Don't create unreadable types** - maintainability matters

---

## Key Takeaways

1. **Conditional types enable compile-time logic**
2. **Mapped types transform object types**
3. **Template literals generate string types**
4. **Recursive types handle nested structures**
5. **Union manipulation creates flexible APIs**
6. **Type-level programming is powerful but complex**
7. **Balance type safety with code readability**
