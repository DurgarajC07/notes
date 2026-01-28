# Template Literal Types in TypeScript

## Core Concept

Template literal types use the same syntax as JavaScript template literals but operate at the **type level**. They enable powerful string manipulation and validation in the type system.

---

## Basic Syntax

```typescript
type World = "world";
type Greeting = `hello ${World}`;
// Type: "hello world"

type EmailLocaleIDs = "welcome_email" | "email_heading";
type FooterLocaleIDs = "footer_title" | "footer_sendoff";

type AllLocaleIDs = `${EmailLocaleIDs | FooterLocaleIDs}_id`;
// Type: "welcome_email_id" | "email_heading_id" | "footer_title_id" | "footer_sendoff_id"
```

---

## String Unions

```typescript
type HTTPMethod = "GET" | "POST" | "PUT" | "DELETE";
type HTTPEndpoint = "/users" | "/posts" | "/comments";

type APIRoute = `${HTTPMethod} ${HTTPEndpoint}`;
// Type: "GET /users" | "GET /posts" | "GET /comments" |
//       "POST /users" | "POST /posts" | "POST /comments" | ...
```

---

## Intrinsic String Manipulation Types

TypeScript provides built-in utility types for string manipulation:

```typescript
type Uppercase<S extends string> // Converts to uppercase
type Lowercase<S extends string> // Converts to lowercase
type Capitalize<S extends string> // Capitalizes first letter
type Uncapitalize<S extends string> // Uncapitalizes first letter

// Examples
type Loud = Uppercase<"hello">;  // "HELLO"
type Quiet = Lowercase<"HELLO">; // "hello"
type Proper = Capitalize<"hello">; // "Hello"
type Lower = Uncapitalize<"Hello">; // "hello"
```

---

## Real-World Example: Event System

```typescript
type EventName = "click" | "focus" | "change";

// Generate event handler types
type EventHandler<E extends EventName> = `on${Capitalize<E>}`;

type ClickHandler = EventHandler<"click">; // "onClick"
type FocusHandler = EventHandler<"focus">; // "onFocus"
type ChangeHandler = EventHandler<"change">; // "onChange"

// Object with all handlers
type EventHandlers = {
  [K in EventName as `on${Capitalize<K>}`]: (event: Event) => void;
};

const handlers: EventHandlers = {
  onClick: (e) => console.log("clicked"),
  onFocus: (e) => console.log("focused"),
  onChange: (e) => console.log("changed"),
};
```

---

## CSS Property Types

```typescript
type CSSProperty = "color" | "background" | "border";
type CSSValue = "red" | "blue" | "green";

type CSSRule = `${CSSProperty}: ${CSSValue}`;
// Type: "color: red" | "color: blue" | ... | "border: green"

// More practical
type Size = "small" | "medium" | "large";
type CSSClass = `btn-${Size}`;
// Type: "btn-small" | "btn-medium" | "btn-large"
```

---

## Database Query Builder

```typescript
type Table = "users" | "posts" | "comments";
type Operation = "select" | "insert" | "update" | "delete";

type Query = `${Operation} * from ${Table}`;
// Type: "select * from users" | "select * from posts" | ...

interface QueryBuilder {
  query: Query;
  execute(): Promise<any>;
}
```

---

## API Endpoint Types

```typescript
type Version = "v1" | "v2";
type Resource = "users" | "products" | "orders";
type Action = "list" | "get" | "create" | "update" | "delete";

type Endpoint = `/api/${Version}/${Resource}/${Action}`;
// Type: "/api/v1/users/list" | "/api/v1/users/get" | ...

function callAPI(endpoint: Endpoint) {
  return fetch(endpoint);
}

callAPI("/api/v1/users/list"); // ✅ Valid
callAPI("/api/v3/users/list"); // ❌ Error
```

---

## Path Parameters

```typescript
type PathParam = `${string}/:${string}`;

function defineRoute(path: PathParam) {
  console.log(`Route: ${path}`);
}

defineRoute("/users/:id"); // ✅
defineRoute("/posts/:postId"); // ✅
defineRoute("/users"); // ❌ Error
```

---

## Type-Safe Object Keys

```typescript
type PropEventSource<T> = {
  on<K extends string & keyof T>(
    eventName: `${K}Changed`,
    callback: (newValue: T[K]) => void,
  ): void;
};

declare function makeWatchedObject<T>(obj: T): T & PropEventSource<T>;

const person = makeWatchedObject({
  firstName: "Homer",
  age: 42,
});

person.on("firstNameChanged", (newName) => {
  console.log(`New name: ${newName.toUpperCase()}`);
});

person.on("ageChanged", (newAge) => {
  console.log(`New age: ${newAge.toFixed()}`);
});

// person.on("emailChanged", ...); // ❌ Error: email doesn't exist
```

---

## Recursive Template Literals

```typescript
type Join<T extends string[], D extends string> = T extends [
  infer F extends string,
  ...infer R extends string[],
]
  ? R["length"] extends 0
    ? F
    : `${F}${D}${Join<R, D>}`
  : "";

type Path = Join<["users", "123", "posts"], "/">;
// Type: "users/123/posts"
```

---

## State Machine Types

```typescript
type State = "idle" | "loading" | "success" | "error";
type Action = "fetch" | "resolve" | "reject" | "reset";

type Transition = `${State}:${Action}`;

type StateMachine = {
  [K in Transition]?: State;
};

const machine: StateMachine = {
  "idle:fetch": "loading",
  "loading:resolve": "success",
  "loading:reject": "error",
  "error:reset": "idle",
  "success:reset": "idle",
};
```

---

## Best Practices

✅ **Use for domain-specific types** (routes, CSS classes, events)  
✅ **Combine with mapped types** for powerful transformations  
✅ **Keep unions small** - large combinations can slow compilation  
✅ **Use intrinsic types** (Uppercase, Capitalize) for readability  
❌ **Don't overuse** - can make types hard to understand  
❌ **Avoid deeply nested** template literals (performance)

---

## Performance Considerations

```typescript
// ⚠️ This creates 1000+ types
type BadExample = `${string}${string}${string}`;

// ✅ Better - constrained unions
type GoodExample = `user-${number}`;
```

---

## Key Takeaways

1. **Template literals work at type level** like JavaScript template literals
2. **Combine with unions** to generate many types from few
3. **Intrinsic types** (Uppercase, etc.) transform strings
4. **Perfect for string patterns** (routes, CSS, events)
5. **Use with mapped types** for powerful transformations
6. **Watch performance** - avoid creating too many types
