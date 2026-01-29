# 🎰 Type-Safe State Machines in TypeScript

> **Model Complex State Transitions with Compile-Time Safety**

---

## 📋 Table of Contents

- [What are State Machines?](#what-are-state-machines)
- [Why Use State Machines?](#why-use-state-machines)
- [Basic Implementation](#basic-implementation)
- [Type-Safe States](#type-safe-states)
- [Discriminated Unions](#discriminated-unions)
- [State Transitions](#state-transitions)
- [XState Integration](#xstate-integration)
- [Real-World Examples](#real-world-examples)
- [Best Practices](#best-practices)

---

## What are State Machines?

A state machine is a mathematical model of computation that describes behavior through states and transitions between those states.

### **Key Concepts**

```typescript
// States: Distinct modes the system can be in
type State = "idle" | "loading" | "success" | "error";

// Events: Actions that trigger transitions
type Event = "FETCH" | "RESOLVE" | "REJECT" | "RETRY";

// Transitions: Rules for moving between states
type Transition = {
  from: State;
  event: Event;
  to: State;
};
```

### **State Machine Properties**

1. **Finite States**: Limited, well-defined states
2. **Deterministic**: Same input always produces same output
3. **Single Active State**: Only one state at a time
4. **Explicit Transitions**: Clear rules for state changes

---

## Why Use State Machines?

### **1. Prevent Invalid States**

```typescript
// ❌ Without state machine - invalid states possible
interface BadAsyncState {
  isLoading: boolean;
  data: User | null;
  error: Error | null;
}

// What does this mean?
const invalid: BadAsyncState = {
  isLoading: true,
  data: { id: "1", name: "John" },
  error: new Error("Failed"),
};

// ✅ With state machine - only valid states
type AsyncState =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; error: Error };

// Can't represent invalid state
const valid: AsyncState = {
  status: "success",
  data: { id: "1", name: "John" },
  // error: new Error() // Type error!
};
```

### **2. Make State Transitions Explicit**

```typescript
// ❌ Implicit transitions - hard to track
function handleClick() {
  if (state.isLoading) return;
  state.isLoading = true;
  // ... somewhere else
  state.isLoading = false;
  state.data = result;
}

// ✅ Explicit transitions - clear flow
function transition(state: State, event: Event): State {
  switch (state.status) {
    case "idle":
      if (event.type === "FETCH") return { status: "loading" };
      return state;

    case "loading":
      if (event.type === "RESOLVE") {
        return { status: "success", data: event.data };
      }
      if (event.type === "REJECT") {
        return { status: "error", error: event.error };
      }
      return state;

    default:
      return state;
  }
}
```

### **3. Self-Documenting Code**

```typescript
// State machine definition IS the documentation
type TrafficLightState =
  | { light: "red"; timer: number }
  | { light: "yellow"; timer: number }
  | { light: "green"; timer: number };

type TrafficLightEvent =
  | { type: "TIMER_END" }
  | { type: "EMERGENCY"; vehicle: "ambulance" | "firetruck" };
```

---

## Basic Implementation

### **Simple State Machine**

```typescript
class StateMachine<S extends string, E extends string> {
  private currentState: S;
  private transitions: Map<string, S>;

  constructor(
    initialState: S,
    transitions: Array<{ from: S; event: E; to: S }>,
  ) {
    this.currentState = initialState;
    this.transitions = new Map();

    transitions.forEach(({ from, event, to }) => {
      const key = `${from}:${event}`;
      this.transitions.set(key, to);
    });
  }

  getState(): S {
    return this.currentState;
  }

  send(event: E): boolean {
    const key = `${this.currentState}:${event}`;
    const nextState = this.transitions.get(key);

    if (nextState) {
      this.currentState = nextState;
      return true;
    }

    return false;
  }
}

// Usage
type DoorState = "closed" | "opening" | "open" | "closing";
type DoorEvent = "OPEN" | "CLOSE" | "COMPLETE";

const door = new StateMachine<DoorState, DoorEvent>("closed", [
  { from: "closed", event: "OPEN", to: "opening" },
  { from: "opening", event: "COMPLETE", to: "open" },
  { from: "open", event: "CLOSE", to: "closing" },
  { from: "closing", event: "COMPLETE", to: "closed" },
]);

console.log(door.getState()); // 'closed'
door.send("OPEN");
console.log(door.getState()); // 'opening'
door.send("COMPLETE");
console.log(door.getState()); // 'open'
```

---

## Type-Safe States

### **Discriminated Union Pattern**

```typescript
type State =
  | { type: "idle" }
  | { type: "loading"; startTime: number }
  | { type: "success"; data: string; timestamp: number }
  | { type: "error"; error: Error; retryCount: number };

function render(state: State) {
  switch (state.type) {
    case "idle":
      return "Ready to fetch";

    case "loading":
      const elapsed = Date.now() - state.startTime;
      return `Loading... (${elapsed}ms)`;

    case "success":
      return `Data: ${state.data} (${state.timestamp})`;

    case "error":
      return `Error: ${state.error.message} (retry #${state.retryCount})`;
  }
}
```

### **With Context Data**

```typescript
interface Context {
  userId: string;
  retries: number;
  history: string[];
}

type State<C = Context> =
  | { status: "idle"; context: C }
  | { status: "loading"; context: C }
  | { status: "success"; context: C; data: User }
  | { status: "error"; context: C; error: Error };

function addToHistory(state: State, message: string): State {
  const newContext = {
    ...state.context,
    history: [...state.context.history, message],
  };

  return { ...state, context: newContext } as State;
}
```

---

## Discriminated Unions

### **Tagged Unions for States**

```typescript
type FormState =
  | { _tag: 'editing'; fields: FormFields; errors: ValidationErrors }
  | { _tag: 'submitting'; fields: FormFields; progress: number }
  | { _tag: 'success'; submittedData: FormData; timestamp: number }
  | { _tag: 'failed'; fields: FormFields; error: Error; canRetry: boolean };

function handleFormState(state: FormState) {
  switch (state._tag) {
    case 'editing':
      return (
        <Form fields={state.fields} errors={state.errors} />
      );

    case 'submitting':
      return (
        <LoadingBar progress={state.progress} />
      );

    case 'success':
      return (
        <SuccessMessage data={state.submittedData} />
      );

    case 'failed':
      return (
        <ErrorMessage
          error={state.error}
          onRetry={state.canRetry ? retry : undefined}
        />
      );
  }
}
```

### **Exhaustive Checking**

```typescript
function assertNever(x: never): never {
  throw new Error(`Unexpected value: ${x}`);
}

function transition(state: State, event: Event): State {
  switch (state.status) {
    case "idle":
      // Handle idle state
      return state;

    case "loading":
      // Handle loading state
      return state;

    case "success":
      // Handle success state
      return state;

    case "error":
      // Handle error state
      return state;

    default:
      // TypeScript ensures all cases handled
      return assertNever(state);
  }
}
```

---

## State Transitions

### **Type-Safe Transitions**

```typescript
type Event =
  | { type: "FETCH" }
  | { type: "RESOLVE"; data: User }
  | { type: "REJECT"; error: Error }
  | { type: "RETRY" }
  | { type: "RESET" };

function transition(state: State, event: Event): State {
  // Pattern match on current state and event
  if (state.status === "idle" && event.type === "FETCH") {
    return { status: "loading" };
  }

  if (state.status === "loading" && event.type === "RESOLVE") {
    return { status: "success", data: event.data };
  }

  if (state.status === "loading" && event.type === "REJECT") {
    return { status: "error", error: event.error };
  }

  if (state.status === "error" && event.type === "RETRY") {
    return { status: "loading" };
  }

  if (event.type === "RESET") {
    return { status: "idle" };
  }

  // No valid transition
  return state;
}
```

### **With Guards**

```typescript
type Guard<S, E> = (state: S, event: E) => boolean;

interface TransitionDef<S, E> {
  from: S["status"];
  event: E["type"];
  to: S["status"];
  guard?: Guard<S, E>;
  action?: (state: S, event: E) => void;
}

function createMachine<
  S extends { status: string },
  E extends { type: string },
>(initialState: S, transitions: TransitionDef<S, E>[]) {
  return function transition(state: S, event: E): S {
    for (const t of transitions) {
      if (
        state.status === t.from &&
        event.type === t.event &&
        (!t.guard || t.guard(state, event))
      ) {
        t.action?.(state, event);
        return { ...state, status: t.to };
      }
    }
    return state;
  };
}

// Usage
const machine = createMachine<State, Event>({ status: "idle" }, [
  {
    from: "idle",
    event: "FETCH",
    to: "loading",
    action: (state, event) => console.log("Starting fetch"),
  },
  {
    from: "error",
    event: "RETRY",
    to: "loading",
    guard: (state) => state.retryCount < 3,
  },
]);
```

---

## XState Integration

### **Basic XState Machine**

```typescript
import { createMachine, interpret, assign } from "xstate";

interface Context {
  retries: number;
  error?: Error;
  data?: User;
}

type Event =
  | { type: "FETCH" }
  | { type: "RESOLVE"; data: User }
  | { type: "REJECT"; error: Error }
  | { type: "RETRY" };

const fetchMachine = createMachine<Context, Event>({
  id: "fetch",
  initial: "idle",
  context: {
    retries: 0,
  },
  states: {
    idle: {
      on: {
        FETCH: "loading",
      },
    },
    loading: {
      on: {
        RESOLVE: {
          target: "success",
          actions: assign({
            data: (_, event) => event.data,
          }),
        },
        REJECT: {
          target: "error",
          actions: assign({
            error: (_, event) => event.error,
            retries: (ctx) => ctx.retries + 1,
          }),
        },
      },
    },
    success: {
      type: "final",
    },
    error: {
      on: {
        RETRY: {
          target: "loading",
          guard: (ctx) => ctx.retries < 3,
        },
      },
    },
  },
});

// Use the machine
const service = interpret(fetchMachine)
  .onTransition((state) => {
    console.log("State:", state.value);
    console.log("Context:", state.context);
  })
  .start();

service.send("FETCH");
service.send({ type: "REJECT", error: new Error("Failed") });
service.send("RETRY");
```

### **With TypeScript**

```typescript
import { createMachine, StateFrom, EventFrom } from "xstate";

const machine = createMachine({
  schema: {
    context: {} as Context,
    events: {} as Event,
  },
  tsTypes: {} as import("./machine.typegen").Typegen0,
  initial: "idle",
  states: {
    idle: {
      on: { FETCH: "loading" },
    },
    loading: {
      on: {
        RESOLVE: "success",
        REJECT: "error",
      },
    },
    success: {},
    error: {
      on: { RETRY: "loading" },
    },
  },
});

// Extract types
type State = StateFrom<typeof machine>;
type Event = EventFrom<typeof machine>;

// Type-safe usage
function handleState(state: State) {
  if (state.matches("idle")) {
    // TypeScript knows we're in idle state
  }

  if (state.matches("success")) {
    // Can access success context
  }
}
```

---

## Real-World Examples

### **Authentication Flow**

```typescript
type AuthState =
  | { status: "unauthenticated" }
  | { status: "authenticating"; provider: string }
  | { status: "authenticated"; user: User; token: string }
  | { status: "error"; error: Error };

type AuthEvent =
  | { type: "LOGIN"; provider: "google" | "github" }
  | { type: "LOGIN_SUCCESS"; user: User; token: string }
  | { type: "LOGIN_FAILURE"; error: Error }
  | { type: "LOGOUT" };

function authTransition(state: AuthState, event: AuthEvent): AuthState {
  switch (state.status) {
    case "unauthenticated":
      if (event.type === "LOGIN") {
        return {
          status: "authenticating",
          provider: event.provider,
        };
      }
      return state;

    case "authenticating":
      if (event.type === "LOGIN_SUCCESS") {
        return {
          status: "authenticated",
          user: event.user,
          token: event.token,
        };
      }
      if (event.type === "LOGIN_FAILURE") {
        return {
          status: "error",
          error: event.error,
        };
      }
      return state;

    case "authenticated":
      if (event.type === "LOGOUT") {
        return { status: "unauthenticated" };
      }
      return state;

    case "error":
      if (event.type === "LOGIN") {
        return {
          status: "authenticating",
          provider: event.provider,
        };
      }
      return state;
  }
}

// React hook
function useAuth() {
  const [state, setState] = useState<AuthState>({
    status: "unauthenticated",
  });

  const send = (event: AuthEvent) => {
    setState((current) => authTransition(current, event));
  };

  return { state, send };
}
```

### **Form Wizard**

```typescript
type WizardState =
  | { step: "personal"; data: PersonalData; errors?: ValidationErrors }
  | { step: "address"; data: AddressData; errors?: ValidationErrors }
  | { step: "payment"; data: PaymentData; errors?: ValidationErrors }
  | { step: "review"; allData: CompleteData }
  | { step: "submitting"; progress: number }
  | { step: "success"; confirmationId: string }
  | { step: "error"; error: Error };

type WizardEvent =
  | { type: "NEXT"; data: any }
  | { type: "BACK" }
  | { type: "SUBMIT" }
  | { type: "SUBMIT_SUCCESS"; confirmationId: string }
  | { type: "SUBMIT_FAILURE"; error: Error };

function wizardTransition(state: WizardState, event: WizardEvent): WizardState {
  switch (state.step) {
    case "personal":
      if (event.type === "NEXT") {
        return {
          step: "address",
          data: {} as AddressData,
        };
      }
      return state;

    case "address":
      if (event.type === "NEXT") {
        return {
          step: "payment",
          data: {} as PaymentData,
        };
      }
      if (event.type === "BACK") {
        return {
          step: "personal",
          data: {} as PersonalData,
        };
      }
      return state;

    case "review":
      if (event.type === "SUBMIT") {
        return {
          step: "submitting",
          progress: 0,
        };
      }
      if (event.type === "BACK") {
        return {
          step: "payment",
          data: {} as PaymentData,
        };
      }
      return state;

    case "submitting":
      if (event.type === "SUBMIT_SUCCESS") {
        return {
          step: "success",
          confirmationId: event.confirmationId,
        };
      }
      if (event.type === "SUBMIT_FAILURE") {
        return {
          step: "error",
          error: event.error,
        };
      }
      return state;

    default:
      return state;
  }
}
```

### **Media Player**

```typescript
type PlayerState =
  | { status: "idle" }
  | { status: "loading"; url: string }
  | { status: "playing"; position: number; duration: number }
  | { status: "paused"; position: number; duration: number }
  | { status: "ended"; duration: number }
  | { status: "error"; error: Error };

type PlayerEvent =
  | { type: "LOAD"; url: string }
  | { type: "LOAD_SUCCESS"; duration: number }
  | { type: "LOAD_FAILURE"; error: Error }
  | { type: "PLAY" }
  | { type: "PAUSE" }
  | { type: "SEEK"; position: number }
  | { type: "ENDED" }
  | { type: "TIME_UPDATE"; position: number };

function playerTransition(state: PlayerState, event: PlayerEvent): PlayerState {
  switch (state.status) {
    case "idle":
      if (event.type === "LOAD") {
        return { status: "loading", url: event.url };
      }
      return state;

    case "loading":
      if (event.type === "LOAD_SUCCESS") {
        return {
          status: "paused",
          position: 0,
          duration: event.duration,
        };
      }
      if (event.type === "LOAD_FAILURE") {
        return { status: "error", error: event.error };
      }
      return state;

    case "paused":
      if (event.type === "PLAY") {
        return {
          status: "playing",
          position: state.position,
          duration: state.duration,
        };
      }
      if (event.type === "SEEK") {
        return { ...state, position: event.position };
      }
      return state;

    case "playing":
      if (event.type === "PAUSE") {
        return {
          status: "paused",
          position: state.position,
          duration: state.duration,
        };
      }
      if (event.type === "TIME_UPDATE") {
        return { ...state, position: event.position };
      }
      if (event.type === "ENDED") {
        return { status: "ended", duration: state.duration };
      }
      return state;

    case "ended":
      if (event.type === "PLAY") {
        return {
          status: "playing",
          position: 0,
          duration: state.duration,
        };
      }
      return state;

    default:
      return state;
  }
}
```

---

## Best Practices

### **1. Use Discriminated Unions**

```typescript
// ✅ Good: Tagged union
type State = { type: "idle" } | { type: "success"; data: string };

// ❌ Bad: Boolean flags
interface BadState {
  isIdle: boolean;
  isSuccess: boolean;
  data?: string;
}
```

### **2. Make Invalid States Unrepresentable**

```typescript
// ✅ Good: Can't have loading with error
type State = { status: "loading" } | { status: "error"; error: Error };

// ❌ Bad: Can have both
interface BadState {
  isLoading: boolean;
  error: Error | null;
}
```

### **3. Keep Transitions Pure**

```typescript
// ✅ Good: Pure function
function transition(state: State, event: Event): State {
  return newState;
}

// ❌ Bad: Side effects
function badTransition(state: State, event: Event): State {
  fetch("/api/endpoint"); // Side effect!
  return newState;
}
```

### **4. Use Context for Shared Data**

```typescript
interface Context {
  userId: string;
  settings: Settings;
}

type State =
  | { status: "idle"; context: Context }
  | { status: "loading"; context: Context };
```

### **5. Exhaustive Matching**

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
    // TypeScript ensures all cases handled
  }
}
```

---

## Summary

### **Benefits**

- ✅ Impossible states become unrepresentable
- ✅ All transitions are explicit and documented
- ✅ Compile-time safety for state handling
- ✅ Easier testing and debugging
- ✅ Better code organization

### **When to Use**

- Complex UI flows (wizards, authentication)
- Async operations with multiple states
- Game state management
- Process orchestration
- Form validation flows

### **Key Principles**

1. Use discriminated unions for states
2. Make transitions explicit
3. Keep state machines pure
4. Leverage TypeScript's type system
5. Consider XState for complex machines

---

**Interview Questions:**

1. What problems do state machines solve?
2. How do you prevent invalid states in TypeScript?
3. What's the difference between state machines and reducers?
4. When would you use XState vs custom implementation?

**Practice:**

- Implement a traffic light state machine
- Build a shopping cart with states
- Create a multi-step form wizard
- Model an authentication flow
