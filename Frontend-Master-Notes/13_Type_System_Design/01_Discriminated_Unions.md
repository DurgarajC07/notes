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
```

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
