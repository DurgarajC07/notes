# Discriminated Unions in TypeScript

## Core Concept

Discriminated unions (also called "tagged unions" or "algebraic data types") use a common property (discriminant) to distinguish between different types in a union, enabling **exhaustive type checking** and **type narrowing**.

---

## Basic Pattern

```typescript
interface Circle {
  kind: "circle"; // Discriminant
  radius: number;
}

interface Square {
  kind: "square"; // Discriminant
  sideLength: number;
}

interface Rectangle {
  kind: "rectangle"; // Discriminant
  width: number;
  height: number;
}

type Shape = Circle | Square | Rectangle;

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2; // TypeScript knows it's Circle
    case "square":
      return shape.sideLength ** 2; // TypeScript knows it's Square
    case "rectangle":
      return shape.width * shape.height; // TypeScript knows it's Rectangle
  }
}
```

---

## Exhaustiveness Checking

```typescript
type Shape = Circle | Square | Rectangle;

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.sideLength ** 2;
    // Missing "rectangle" case!
    default:
      const _exhaustive: never = shape; // ❌ Error if not exhaustive
      return _exhaustive;
  }
}
```

---

## API Response Pattern

```typescript
interface LoadingState {
  status: "loading";
}

interface SuccessState<T> {
  status: "success";
  data: T;
}

interface ErrorState {
  status: "error";
  error: string;
}

type AsyncState<T> = LoadingState | SuccessState<T> | ErrorState;

function handleResponse<T>(state: AsyncState<T>) {
  switch (state.status) {
    case "loading":
      return "Loading...";
    case "success":
      return state.data; // Type: T (knows data exists)
    case "error":
      return state.error; // Type: string (knows error exists)
  }
}

// Usage
const userState: AsyncState<User> = { status: "success", data: user };
const result = handleResponse(userState); // Type: User | string | "Loading..."
```

---

## Redux Action Pattern

```typescript
interface IncrementAction {
  type: "INCREMENT";
}

interface DecrementAction {
  type: "DECREMENT";
}

interface SetCountAction {
  type: "SET_COUNT";
  payload: number;
}

type CounterAction = IncrementAction | DecrementAction | SetCountAction;

function reducer(state: number, action: CounterAction): number {
  switch (action.type) {
    case "INCREMENT":
      return state + 1;
    case "DECREMENT":
      return state - 1;
    case "SET_COUNT":
      return action.payload; // TypeScript knows payload exists
    default:
      const _exhaustive: never = action;
      return state;
  }
}
```

---

## Form Validation Result

```typescript
interface Valid<T> {
  valid: true;
  value: T;
}

interface Invalid {
  valid: false;
  errors: string[];
}

type ValidationResult<T> = Valid<T> | Invalid;

function validateEmail(email: string): ValidationResult<string> {
  if (email.includes("@")) {
    return { valid: true, value: email };
  }
  return { valid: false, errors: ["Invalid email format"] };
}

function handleValidation(result: ValidationResult<string>) {
  if (result.valid) {
    console.log(`Valid email: ${result.value}`); // Type: string
  } else {
    console.log(`Errors: ${result.errors.join(", ")}`); // Type: string[]
  }
}
```

---

## WebSocket Message Types

```typescript
interface ConnectMessage {
  type: "CONNECT";
  userId: string;
}

interface DisconnectMessage {
  type: "DISCONNECT";
  reason: string;
}

interface ChatMessage {
  type: "CHAT";
  message: string;
  timestamp: number;
}

type WebSocketMessage = ConnectMessage | DisconnectMessage | ChatMessage;

function handleMessage(msg: WebSocketMessage) {
  switch (msg.type) {
    case "CONNECT":
      console.log(`User ${msg.userId} connected`);
      break;
    case "DISCONNECT":
      console.log(`Disconnected: ${msg.reason}`);
      break;
    case "CHAT":
      console.log(`[${msg.timestamp}] ${msg.message}`);
      break;
  }
}
```

---

## Payment Method Pattern

```typescript
interface CreditCard {
  method: "credit_card";
  cardNumber: string;
  cvv: string;
  expiry: string;
}

interface PayPal {
  method: "paypal";
  email: string;
}

interface BankTransfer {
  method: "bank_transfer";
  accountNumber: string;
  routingNumber: string;
}

type PaymentMethod = CreditCard | PayPal | BankTransfer;

function processPayment(payment: PaymentMethod, amount: number) {
  switch (payment.method) {
    case "credit_card":
      return chargeCreditCard(payment.cardNumber, payment.cvv, amount);
    case "paypal":
      return chargePayPal(payment.email, amount);
    case "bank_transfer":
      return initiateBankTransfer(payment.accountNumber, amount);
  }
}
```

---

## Generic Discriminated Union

```typescript
interface Success<T> {
  status: "success";
  value: T;
}

interface Failure<E> {
  status: "failure";
  error: E;
}

type Result<T, E = Error> = Success<T> | Failure<E>;

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) {
    return { status: "failure", error: "Division by zero" };
  }
  return { status: "success", value: a / b };
}

const result = divide(10, 2);
if (result.status === "success") {
  console.log(result.value); // Type: number
} else {
  console.log(result.error); // Type: string
}
```

---

## Multiple Discriminants

```typescript
interface Dog {
  species: "dog";
  breed: "labrador" | "poodle";
  bark: () => void;
}

interface Cat {
  species: "cat";
  breed: "persian" | "siamese";
  meow: () => void;
}

type Pet = Dog | Cat;

function makeSound(pet: Pet) {
  if (pet.species === "dog") {
    if (pet.breed === "labrador") {
      console.log("Big woof!");
    }
    pet.bark(); // TypeScript knows it's Dog
  } else {
    pet.meow(); // TypeScript knows it's Cat
  }
}
```

---

## React Component State Pattern

```typescript
interface IdleState {
  status: "idle";
}

interface FetchingState {
  status: "fetching";
  controller: AbortController;
}

interface SuccessState<T> {
  status: "success";
  data: T;
  timestamp: number;
}

interface ErrorState {
  status: "error";
  error: Error;
  retry: () => void;
}

type FetchState<T> = IdleState | FetchingState | SuccessState<T> | ErrorState;

interface UserData {
  id: string;
  name: string;
}

function UserComponent({ state }: { state: FetchState<UserData> }) {
  switch (state.status) {
    case "idle":
      return <button>Load User</button>;

    case "fetching":
      return (
        <div>
          Loading...
          <button onClick={() => state.controller.abort()}>Cancel</button>
        </div>
      );

    case "success":
      return (
        <div>
          <h2>{state.data.name}</h2>
          <small>Loaded at {new Date(state.timestamp).toLocaleString()}</small>
        </div>
      );

    case "error":
      return (
        <div>
          <p>Error: {state.error.message}</p>
          <button onClick={state.retry}>Retry</button>
        </div>
      );
  }
}
```

---

## File Upload Progress Pattern

```typescript
interface NotStarted {
  stage: "not-started";
}

interface Uploading {
  stage: "uploading";
  progress: number; // 0-100
  bytesUploaded: number;
  totalBytes: number;
}

interface Processing {
  stage: "processing";
  message: string;
}

interface Completed {
  stage: "completed";
  fileUrl: string;
  thumbnailUrl?: string;
}

interface Failed {
  stage: "failed";
  error: string;
  canRetry: boolean;
}

type UploadState = NotStarted | Uploading | Processing | Completed | Failed;

function renderUploadUI(state: UploadState) {
  switch (state.stage) {
    case "not-started":
      return "Select a file to upload";

    case "uploading":
      const percentage = Math.round(
        (state.bytesUploaded / state.totalBytes) * 100,
      );
      return `Uploading... ${percentage}% (${state.bytesUploaded}/${state.totalBytes} bytes)`;

    case "processing":
      return `Processing: ${state.message}`;

    case "completed":
      return `Upload complete! File URL: ${state.fileUrl}`;

    case "failed":
      return `Upload failed: ${state.error}. ${state.canRetry ? "Click to retry" : "Cannot retry"}`;
  }
}
```

---

## Authentication State Pattern

```typescript
interface Unauthenticated {
  auth: "unauthenticated";
}

interface Authenticating {
  auth: "authenticating";
  method: "password" | "oauth" | "biometric";
}

interface Authenticated {
  auth: "authenticated";
  user: {
    id: string;
    email: string;
    roles: string[];
  };
  token: string;
  expiresAt: number;
}

interface AuthError {
  auth: "error";
  reason: string;
  needsVerification?: boolean;
}

type AuthState = Unauthenticated | Authenticating | Authenticated | AuthError;

function getAuthStatus(state: AuthState): string {
  switch (state.auth) {
    case "unauthenticated":
      return "Please log in";

    case "authenticating":
      return `Authenticating via ${state.method}...`;

    case "authenticated":
      const isExpired = Date.now() > state.expiresAt;
      return isExpired ? "Session expired" : `Logged in as ${state.user.email}`;

    case "error":
      return state.needsVerification
        ? `${state.reason}. Please verify your email.`
        : state.reason;
  }
}
```

---

## Database Query Result Pattern

```typescript
interface EmptyResult {
  type: "empty";
  query: string;
}

interface SingleResult<T> {
  type: "single";
  data: T;
}

interface MultipleResults<T> {
  type: "multiple";
  data: T[];
  total: number;
  hasMore: boolean;
}

interface QueryError {
  type: "error";
  code: string;
  message: string;
}

type QueryResult<T> =
  | EmptyResult
  | SingleResult<T>
  | MultipleResults<T>
  | QueryError;

function handleQueryResult<T>(result: QueryResult<T>) {
  switch (result.type) {
    case "empty":
      console.log(`No results for query: ${result.query}`);
      return null;

    case "single":
      console.log("Found one result:", result.data);
      return result.data;

    case "multiple":
      console.log(
        `Found ${result.data.length} results (total: ${result.total})`,
      );
      if (result.hasMore) {
        console.log("More results available");
      }
      return result.data;

    case "error":
      console.error(`Query error [${result.code}]: ${result.message}`);
      return null;
  }
}
```

---

## Notification Types Pattern

```typescript
interface InfoNotification {
  level: "info";
  message: string;
  icon?: string;
}

interface WarningNotification {
  level: "warning";
  message: string;
  dismissible: boolean;
}

interface ErrorNotification {
  level: "error";
  message: string;
  error: Error;
  retryAction?: () => void;
}

interface SuccessNotification {
  level: "success";
  message: string;
  duration: number;
  action?: {
    label: string;
    onClick: () => void;
  };
}

type Notification =
  | InfoNotification
  | WarningNotification
  | ErrorNotification
  | SuccessNotification;

function showNotification(notification: Notification) {
  const baseStyle = { padding: "12px", borderRadius: "4px" };

  switch (notification.level) {
    case "info":
      return {
        ...baseStyle,
        background: "#e3f2fd",
        icon: notification.icon || "ℹ️",
        text: notification.message,
      };

    case "warning":
      return {
        ...baseStyle,
        background: "#fff3e0",
        text: notification.message,
        showClose: notification.dismissible,
      };

    case "error":
      return {
        ...baseStyle,
        background: "#ffebee",
        text: notification.message,
        details: notification.error.stack,
        retryButton: notification.retryAction,
      };

    case "success":
      return {
        ...baseStyle,
        background: "#e8f5e9",
        text: notification.message,
        autoHide: notification.duration,
        actionButton: notification.action,
      };
  }
}
```

---

## Navigation Event Pattern

```typescript
interface PushEvent {
  type: "push";
  path: string;
  state?: Record<string, unknown>;
}

interface ReplaceEvent {
  type: "replace";
  path: string;
}

interface PopEvent {
  type: "pop";
  delta: number;
}

interface ReloadEvent {
  type: "reload";
  forceFresh: boolean;
}

type NavigationEvent = PushEvent | ReplaceEvent | PopEvent | ReloadEvent;

function handleNavigation(event: NavigationEvent) {
  switch (event.type) {
    case "push":
      window.history.pushState(event.state, "", event.path);
      break;

    case "replace":
      window.history.replaceState(null, "", event.path);
      break;

    case "pop":
      window.history.go(event.delta);
      break;

    case "reload":
      if (event.forceFresh) {
        window.location.reload();
      } else {
        // Soft reload
      }
      break;
  }
}
```

---

## Best Practices

### ✅ DO: Use Literal Types as Discriminants

```typescript
// Good - string literals
type Status = "idle" | "loading" | "success" | "error";

interface State {
  status: Status;
}

// Good - numeric literals
type HttpStatus = 200 | 404 | 500;
```

### ❌ DON'T: Use Non-Literal Types

```typescript
// Bad - TypeScript can't narrow properly
interface State {
  status: string; // Too broad
}
```

### ✅ DO: Use Exhaustiveness Checking

```typescript
function handle(state: State): string {
  switch (state.status) {
    case "idle":
      return "Idle";
    case "loading":
      return "Loading";
    case "success":
      return "Success";
    case "error":
      return "Error";
    default:
      const _exhaustive: never = state; // Compiler enforces all cases
      return _exhaustive;
  }
}
```

### ✅ DO: Keep Discriminant Names Consistent

```typescript
// Good - all use "type"
interface ActionA {
  type: "A";
}
interface ActionB {
  type: "B";
}

// Bad - mixed discriminant names
interface ActionA {
  type: "A";
}
interface ActionB {
  kind: "B";
} // Different property name!
```

---

## Real-World Production Example

```typescript
// API Response Wrapper
interface ApiSuccess<T> {
  status: "success";
  data: T;
  metadata: {
    requestId: string;
    timestamp: number;
  };
}

interface ApiError {
  status: "error";
  error: {
    code: string;
    message: string;
    details?: Record<string, unknown>;
  };
  retry: {
    canRetry: boolean;
    retryAfter?: number;
  };
}

type ApiResponse<T> = ApiSuccess<T> | ApiError;

// Type-safe API client
async function fetchUser(id: string): Promise<ApiResponse<User>> {
  try {
    const response = await fetch(`/api/users/${id}`);
    const data = await response.json();

    if (!response.ok) {
      return {
        status: "error",
        error: {
          code: `HTTP_${response.status}`,
          message: data.message || "Request failed",
        },
        retry: {
          canRetry: response.status >= 500,
          retryAfter: parseInt(response.headers.get("Retry-After") || "0"),
        },
      };
    }

    return {
      status: "success",
      data: data,
      metadata: {
        requestId: response.headers.get("X-Request-ID") || "",
        timestamp: Date.now(),
      },
    };
  } catch (error) {
    return {
      status: "error",
      error: {
        code: "NETWORK_ERROR",
        message: error instanceof Error ? error.message : "Unknown error",
      },
      retry: {
        canRetry: true,
      },
    };
  }
}

// Usage with exhaustive handling
async function loadUser(id: string) {
  const result = await fetchUser(id);

  switch (result.status) {
    case "success":
      console.log(`User loaded: ${result.data.name}`);
      console.log(`Request ID: ${result.metadata.requestId}`);
      return result.data;

    case "error":
      console.error(`Error [${result.error.code}]: ${result.error.message}`);

      if (result.retry.canRetry) {
        if (result.retry.retryAfter) {
          console.log(`Retry after ${result.retry.retryAfter} seconds`);
          await new Promise((resolve) =>
            setTimeout(resolve, result.retry.retryAfter! * 1000),
          );
          return loadUser(id); // Retry
        }
      }

      throw new Error(result.error.message);
  }
}
```

---

## Key Takeaways

1. **Type Safety**: Discriminated unions provide compile-time guarantees that all cases are handled
2. **Exhaustiveness**: Use `never` type in default case to catch missing branches
3. **Narrow Types**: TypeScript automatically narrows types based on discriminant checks
4. **Consistent Discriminants**: Use the same property name across all union members
5. **Literal Types**: Use string/numeric literals as discriminants, not broad types
6. **Generic Support**: Discriminated unions work well with generic types
7. **Multiple Discriminants**: You can use multiple properties for more complex narrowing
8. **Production Pattern**: Ideal for API responses, state machines, event handling
9. **Redux/State Management**: Perfect pattern for action types and state transitions
10. **Error Handling**: Makes impossible states truly impossible to represent
    pet.meow(); // TypeScript knows it's Cat
    }
    }

````

---

## Helper Function Pattern

```typescript
type Shape = Circle | Square | Rectangle;

function isCircle(shape: Shape): shape is Circle {
  return shape.kind === "circle";
}

function isSquare(shape: Shape): shape is Square {
  return shape.kind === "square";
}

function area(shape: Shape): number {
  if (isCircle(shape)) {
    return Math.PI * shape.radius ** 2;
  }
  if (isSquare(shape)) {
    return shape.sideLength ** 2;
  }
  return shape.width * shape.height;
}
````

---

## Best Practices

✅ **Use literal types** for discriminants (string literals, not string)  
✅ **Name discriminants consistently** (type, kind, status)  
✅ **Always include exhaustiveness check** with never  
✅ **Use discriminated unions** for state machines  
✅ **Keep discriminants required** (not optional)  
❌ **Don't use boolean discriminants** - hard to extend  
❌ **Don't mix discriminant names** - inconsistent

---

## Common Discriminant Names

- `type` - For action types, message types
- `kind` - For shape types, node types
- `status` - For async states, request states
- `variant` - For UI component variants
- `tag` - Generic discriminant

---

## Key Takeaways

1. **Discriminant property** uniquely identifies union member
2. **Enables exhaustive checking** with never type
3. **TypeScript narrows types** automatically in switch/if
4. **Perfect for state machines** and Redux actions
5. **Use literal types** for discriminants, not primitives
6. **Generic discriminated unions** provide reusability
