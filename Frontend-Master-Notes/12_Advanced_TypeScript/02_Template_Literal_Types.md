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

## Advanced Pattern: Type-Safe Router

```typescript
type HTTPMethod = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";
type RouteParam = `:${string}`;

type Route = `/api/${string}`;
type RouteWithParam = `/api/${string}/:${string}`;

interface RouteDefinition<Path extends string> {
  path: Path;
  method: HTTPMethod;
  handler: (req: Request) => Response;
}

// Extract params from path
type ExtractParams<Path extends string> =
  Path extends `${infer _Start}/:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof ExtractParams<`/${Rest}`>]: string }
    : Path extends `${infer _Start}/:${infer Param}`
      ? { [K in Param]: string }
      : {};

type UserByIdParams = ExtractParams<"/api/users/:userId">;
// Type: { userId: string }

type PostCommentParams =
  ExtractParams<"/api/posts/:postId/comments/:commentId">;
// Type: { postId: string; commentId: string }

// Type-safe route handler
function defineRoute<Path extends string>(
  path: Path,
  handler: (params: ExtractParams<Path>) => void,
) {
  return { path, handler };
}

// Usage - params are typed!
const userRoute = defineRoute("/api/users/:userId", (params) => {
  console.log(params.userId); // ✅ Type-safe
  // console.log(params.postId); // ❌ Error
});

const postRoute = defineRoute(
  "/api/posts/:postId/comments/:commentId",
  (params) => {
    console.log(params.postId, params.commentId); // ✅ Both available
  },
);
```

---

## Redux Action Types

```typescript
type Entity = "user" | "post" | "comment";
type Operation = "fetch" | "create" | "update" | "delete";
type Status = "request" | "success" | "failure";

type ActionType = `${Entity}/${Operation}/${Status}`;

// Generates: "user/fetch/request", "user/fetch/success", "user/fetch/failure", ...
// Total: 3 entities × 4 operations × 3 statuses = 36 action types

interface Action<Type extends ActionType, Payload = void> {
  type: Type;
  payload: Payload;
}

type UserFetchRequest = Action<"user/fetch/request", { userId: string }>;
type UserFetchSuccess = Action<"user/fetch/success", { user: User }>;
type UserFetchFailure = Action<"user/fetch/failure", { error: string }>;

// Type-safe action creators
function createAction<Type extends ActionType, Payload>(
  type: Type,
): (payload: Payload) => Action<Type, Payload> {
  return (payload) => ({ type, payload });
}

const fetchUserRequest = createAction<"user/fetch/request", { userId: string }>(
  "user/fetch/request",
);
const fetchUserSuccess = createAction<"user/fetch/success", { user: User }>(
  "user/fetch/success",
);

// Usage
const action = fetchUserRequest({ userId: "123" });
// Type: Action<'user/fetch/request', { userId: string }>
```

---

## CSS-in-JS Type Safety

```typescript
type CSSUnit = "px" | "em" | "rem" | "%" | "vh" | "vw";
type CSSValue<U extends CSSUnit> = `${number}${U}`;

type Spacing = CSSValue<"px"> | CSSValue<"rem">;
type FontSize = CSSValue<"px"> | CSSValue<"rem"> | CSSValue<"em">;

interface Theme {
  spacing: {
    small: Spacing;
    medium: Spacing;
    large: Spacing;
  };
  fontSize: {
    body: FontSize;
    heading: FontSize;
  };
}

const theme: Theme = {
  spacing: {
    small: "8px",
    medium: "16px",
    large: "24px",
  },
  fontSize: {
    body: "16px",
    heading: "2rem",
  },
};

// Generate CSS custom properties
type ThemeToCSSVars<T> = {
  [K in keyof T as `--${string & K}`]: T[K] extends object
    ? ThemeToCSSVars<T[K]>
    : string;
};

type CSSVars = ThemeToCSSVars<Theme>;
// Type: {
//   '--spacing': { '--small': string; '--medium': string; '--large': string };
//   '--fontSize': { '--body': string; '--heading': string };
// }
```

---

## Environment Variable Typing

```typescript
type EnvPrefix = "REACT_APP" | "NEXT_PUBLIC" | "VITE";
type EnvVar<
  Prefix extends EnvPrefix,
  Name extends string,
> = `${Prefix}_${Uppercase<Name>}`;

type ReactEnvVar<Name extends string> = EnvVar<"REACT_APP", Name>;

type AppConfig = {
  [K in ReactEnvVar<"apiUrl" | "apiKey" | "environment">]: string;
};

// Usage
declare const process: {
  env: AppConfig & Record<string, string | undefined>;
};

const apiUrl = process.env.REACT_APP_API_URL; // ✅ Type: string
const apiKey = process.env.REACT_APP_API_KEY; // ✅ Type: string
// const wrong = process.env.API_URL; // ❌ Not in type
```

---

## GraphQL Query Types

```typescript
type Operation = "query" | "mutation" | "subscription";
type OperationName = string;

type GraphQLOperation = `${Operation} ${OperationName}`;

interface QueryDefinition<Op extends GraphQLOperation> {
  operation: Op;
  variables?: Record<string, any>;
}

function defineQuery<Op extends GraphQLOperation>(
  operation: Op,
  variables?: Record<string, any>,
): QueryDefinition<Op> {
  return { operation, variables };
}

// Usage - enforces operation format
const getUserQuery = defineQuery("query GetUser", { userId: "123" });
const createPostMutation = defineQuery("mutation CreatePost", {
  title: "Hello",
});

// const invalid = defineQuery('load GetUser'); // ❌ Error
```

---

## File Path Types

```typescript
type FileExtension = "ts" | "tsx" | "js" | "jsx" | "json" | "css" | "scss";
type FileName = `${string}.${FileExtension}`;

type Directory = "src" | "public" | "config" | "tests";
type FilePath = `${Directory}/${FileName}`;

function importFile(path: FilePath) {
  console.log(`Importing: ${path}`);
}

importFile("src/App.tsx"); // ✅
importFile("config/env.json"); // ✅
importFile("assets/logo.png"); // ❌ Error - 'assets' not in Directory
importFile("src/App.ts"); // ✅
importFile("src/App"); // ❌ Error - missing extension
```

---

## Event Emitter Pattern

```typescript
type EventMap = {
  "user:login": { userId: string; timestamp: number };
  "user:logout": { userId: string };
  "post:created": { postId: string; authorId: string };
  "post:updated": { postId: string; changes: string[] };
};

type EventName = keyof EventMap;
type EventHandler<E extends EventName> = (data: EventMap[E]) => void;

class TypedEventEmitter {
  private listeners: {
    [E in EventName]?: EventHandler<E>[];
  } = {};

  on<E extends EventName>(event: E, handler: EventHandler<E>) {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event]!.push(handler);
  }

  emit<E extends EventName>(event: E, data: EventMap[E]) {
    const handlers = this.listeners[event];
    if (handlers) {
      handlers.forEach((handler) => handler(data));
    }
  }
}

// Usage - fully typed!
const emitter = new TypedEventEmitter();

emitter.on("user:login", (data) => {
  console.log(`User ${data.userId} logged in at ${data.timestamp}`);
  // data is typed as { userId: string; timestamp: number }
});

emitter.emit("user:login", { userId: "123", timestamp: Date.now() }); // ✅
// emitter.emit('user:login', { userId: '123' }); // ❌ Error - missing timestamp
```

---

## Validation Schema Types

```typescript
type Validator = "required" | "email" | "min" | "max" | "pattern";
type ValidationRule = `${Validator}:${string}` | Validator;

interface FieldSchema {
  name: string;
  type: "string" | "number" | "boolean";
  validations: ValidationRule[];
}

const userSchema: FieldSchema[] = [
  {
    name: "email",
    type: "string",
    validations: ["required", "email"],
  },
  {
    name: "age",
    type: "number",
    validations: ["required", "min:18", "max:120"],
  },
  {
    name: "username",
    type: "string",
    validations: ["required", "pattern:[a-zA-Z0-9]+"],
  },
];
```

---

## Database Column Types

```typescript
type DataType = "VARCHAR" | "INT" | "BOOLEAN" | "TIMESTAMP";
type ColumnConstraint = "NOT NULL" | "UNIQUE" | "PRIMARY KEY";

type ColumnDefinition =
  | `${string} ${DataType}`
  | `${string} ${DataType} ${ColumnConstraint}`;

type CreateTableSQL = `CREATE TABLE ${string} (${ColumnDefinition})`;

const createUsersTable: CreateTableSQL =
  "CREATE TABLE users (id INT PRIMARY KEY)";

const createPostsTable: CreateTableSQL =
  "CREATE TABLE posts (title VARCHAR NOT NULL)";
```

---

## Branded String Types

```typescript
type Brand<T, TBrand> = T & { __brand: TBrand };

type UserId = Brand<string, "UserId">;
type Email = Brand<string, "Email">;
type URL = Brand<string, "URL">;

// Type-safe constructors
function createUserId(id: string): UserId {
  return id as UserId;
}

function createEmail(email: string): Email {
  if (!email.includes("@")) {
    throw new Error("Invalid email");
  }
  return email as Email;
}

function createURL(url: string): URL {
  try {
    new URL(url);
    return url as URL;
  } catch {
    throw new Error("Invalid URL");
  }
}

// Usage - cannot mix types
const userId = createUserId("user-123");
const email = createEmail("user@example.com");

function sendEmail(to: Email, from: Email) {
  console.log(`Sending email from ${from} to ${to}`);
}

sendEmail(email, email); // ✅
// sendEmail(userId, email); // ❌ Error - UserId is not Email
```

---

## Performance Considerations

```typescript
// ⚠️ Bad - Creates 1000+ type combinations (slow compilation)
type BadExample = `${string}${string}${string}`;

// ⚠️ Bad - Too many combinations
type HTTPMethod =
  | "GET"
  | "POST"
  | "PUT"
  | "DELETE"
  | "PATCH"
  | "HEAD"
  | "OPTIONS";
type Version = "v1" | "v2" | "v3" | "v4" | "v5";
type Resource = "users" | "posts" | "comments" | "likes" | "shares" | "follows";
type AllEndpoints = `/${Version}/${Resource}`; // 7 × 5 × 6 = 210 combinations

// ✅ Good - Constrained unions
type HTTPMethod = "GET" | "POST" | "PUT" | "DELETE";
type Endpoint = `/api/users` | `/api/posts`;
type Route = `${HTTPMethod} ${Endpoint}`; // 4 × 2 = 8 combinations

// ✅ Good - Use generic parameters instead
function makeRequest<Path extends string>(path: Path): Promise<Response> {
  return fetch(path);
}
```

---

## Key Takeaways

1. **Template literals work at type level** like JavaScript template literals
2. **Combine with unions** to generate many types from few
3. **Intrinsic types** (Uppercase, Lowercase, Capitalize, Uncapitalize) transform strings
4. **Extract pattern matching** enables parsing structured strings
5. **Type-safe APIs** prevent runtime errors by validating at compile time
6. **Redux/Event systems** benefit from exhaustive action/event typing
7. **CSS-in-JS** can enforce valid property values and units
8. **Environment variables** can be typed for better DX
9. **Keep unions small** - large combinations slow compilation
10. **Branded types** with template literals prevent mixing incompatible strings
11. **Intrinsic types** (Uppercase, etc.) transform strings
12. **Perfect for string patterns** (routes, CSS, events)
13. **Use with mapped types** for powerful transformations
14. **Watch performance** - avoid creating too many types
