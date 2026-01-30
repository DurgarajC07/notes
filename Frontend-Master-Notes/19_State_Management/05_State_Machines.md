# State Machines with XState

## Core Concept

State machines explicitly model all possible states and transitions in your application. XState implements hierarchical state machines (statecharts) with TypeScript support, making complex state logic predictable and testable.

---

## Basic State Machine

```typescript
import { createMachine, interpret } from "xstate";

// Traffic light state machine
const trafficLightMachine = createMachine({
  id: "trafficLight",
  initial: "green",
  states: {
    green: {
      after: {
        3000: "yellow", // Auto-transition after 3s
      },
    },
    yellow: {
      after: {
        1000: "red",
      },
    },
    red: {
      after: {
        3000: "green",
      },
    },
  },
});

// Usage
const service = interpret(trafficLightMachine)
  .onTransition((state) => {
    console.log("Current state:", state.value);
  })
  .start();
```

---

## Events and Transitions

```typescript
const toggleMachine = createMachine({
  id: "toggle",
  initial: "inactive",
  states: {
    inactive: {
      on: {
        TOGGLE: "active",
      },
    },
    active: {
      on: {
        TOGGLE: "inactive",
      },
    },
  },
});

// Send events
service.send("TOGGLE"); // inactive -> active
service.send("TOGGLE"); // active -> inactive
```

---

## Context (Extended State)

```typescript
interface CounterContext {
  count: number;
}

type CounterEvent =
  | { type: "INCREMENT" }
  | { type: "DECREMENT" }
  | { type: "RESET" };

const counterMachine = createMachine<CounterContext, CounterEvent>({
  id: "counter",
  initial: "active",
  context: {
    count: 0,
  },
  states: {
    active: {
      on: {
        INCREMENT: {
          actions: assign({
            count: (context) => context.count + 1,
          }),
        },
        DECREMENT: {
          actions: assign({
            count: (context) => context.count - 1,
          }),
        },
        RESET: {
          actions: assign({
            count: 0,
          }),
        },
      },
    },
  },
});
```

---

## Guards (Conditional Transitions)

```typescript
const authMachine = createMachine({
  id: "auth",
  initial: "loggedOut",
  context: {
    attempts: 0,
    maxAttempts: 3,
  },
  states: {
    loggedOut: {
      on: {
        LOGIN: {
          target: "loggingIn",
        },
      },
    },
    loggingIn: {
      invoke: {
        src: "loginService",
        onDone: {
          target: "loggedIn",
          actions: assign({ attempts: 0 }),
        },
        onError: [
          {
            target: "locked",
            cond: (context) => context.attempts >= context.maxAttempts,
            actions: assign({ attempts: (ctx) => ctx.attempts + 1 }),
          },
          {
            target: "loggedOut",
            actions: assign({ attempts: (ctx) => ctx.attempts + 1 }),
          },
        ],
      },
    },
    loggedIn: {
      on: {
        LOGOUT: "loggedOut",
      },
    },
    locked: {
      after: {
        60000: "loggedOut", // Unlock after 1 minute
      },
    },
  },
});
```

---

## Hierarchical States

```typescript
const playerMachine = createMachine({
  id: "player",
  initial: "stopped",
  states: {
    stopped: {
      on: {
        PLAY: "playing",
      },
    },
    playing: {
      initial: "normal",
      states: {
        normal: {
          on: {
            SPEED_UP: "fast",
          },
        },
        fast: {
          on: {
            SLOW_DOWN: "normal",
          },
        },
      },
      on: {
        PAUSE: "paused",
        STOP: "stopped",
      },
    },
    paused: {
      on: {
        PLAY: "playing",
        STOP: "stopped",
      },
    },
  },
});
```

---

## Parallel States

```typescript
const appMachine = createMachine({
  id: "app",
  type: "parallel",
  states: {
    network: {
      initial: "offline",
      states: {
        offline: {
          on: {
            CONNECT: "online",
          },
        },
        online: {
          on: {
            DISCONNECT: "offline",
          },
        },
      },
    },
    theme: {
      initial: "light",
      states: {
        light: {
          on: {
            TOGGLE_THEME: "dark",
          },
        },
        dark: {
          on: {
            TOGGLE_THEME: "light",
          },
        },
      },
    },
  },
});

// Can be in both states: { network: 'online', theme: 'dark' }
```

---

## Services (Invoked Promises)

```typescript
interface UserContext {
  user: User | null;
  error: Error | null;
}

const userMachine = createMachine<UserContext>({
  id: "user",
  initial: "idle",
  context: {
    user: null,
    error: null,
  },
  states: {
    idle: {
      on: {
        FETCH: "loading",
      },
    },
    loading: {
      invoke: {
        src: async (context, event) => {
          const response = await fetch(`/api/users/${event.userId}`);
          if (!response.ok) throw new Error("Failed to fetch");
          return response.json();
        },
        onDone: {
          target: "success",
          actions: assign({
            user: (context, event) => event.data,
          }),
        },
        onError: {
          target: "failure",
          actions: assign({
            error: (context, event) => event.data,
          }),
        },
      },
    },
    success: {
      on: {
        FETCH: "loading",
      },
    },
    failure: {
      on: {
        RETRY: "loading",
      },
    },
  },
});
```

---

## XState with React

```typescript
import { useMachine } from '@xstate/react';

function ToggleButton() {
  const [state, send] = useMachine(toggleMachine);

  return (
    <button onClick={() => send('TOGGLE')}>
      {state.value === 'active' ? 'Active' : 'Inactive'}
    </button>
  );
}

// With context
function Counter() {
  const [state, send] = useMachine(counterMachine);

  return (
    <div>
      <div>Count: {state.context.count}</div>
      <button onClick={() => send('INCREMENT')}>+</button>
      <button onClick={() => send('DECREMENT')}>-</button>
      <button onClick={() => send('RESET')}>Reset</button>
    </div>
  );
}
```

---

## Real-World: Form Validation

```typescript
interface FormContext {
  name: string;
  email: string;
  errors: Record<string, string>;
}

type FormEvent =
  | { type: 'UPDATE_NAME'; value: string }
  | { type: 'UPDATE_EMAIL'; value: string }
  | { type: 'SUBMIT' }
  | { type: 'RESET' };

const formMachine = createMachine<FormContext, FormEvent>({
  id: 'form',
  initial: 'editing',
  context: {
    name: '',
    email: '',
    errors: {}
  },
  states: {
    editing: {
      on: {
        UPDATE_NAME: {
          actions: assign({
            name: (_, event) => event.value
          })
        },
        UPDATE_EMAIL: {
          actions: assign({
            email: (_, event) => event.value
          })
        },
        SUBMIT: {
          target: 'validating'
        }
      }
    },
    validating: {
      invoke: {
        src: async (context) => {
          const errors: Record<string, string> = {};

          if (!context.name) {
            errors.name = 'Name is required';
          }

          if (!context.email.includes('@')) {
            errors.email = 'Invalid email';
          }

          if (Object.keys(errors).length > 0) {
            throw errors;
          }

          return context;
        },
        onDone: {
          target: 'submitting',
          actions: assign({ errors: {} })
        },
        onError: {
          target: 'editing',
          actions: assign({
            errors: (_, event) => event.data
          })
        }
      }
    },
    submitting: {
      invoke: {
        src: async (context) => {
          const response = await fetch('/api/submit', {
            method: 'POST',
            body: JSON.stringify(context)
          });

          if (!response.ok) throw new Error('Submit failed');
          return response.json();
        },
        onDone: 'success',
        onError: {
          target: 'editing',
          actions: assign({
            errors: { submit: 'Submission failed' }
          })
        }
      }
    },
    success: {
      on: {
        RESET: {
          target: 'editing',
          actions: assign({
            name: '',
            email: '',
            errors: {}
          })
        }
      }
    }
  }
});

// Component
function FormComponent() {
  const [state, send] = useMachine(formMachine);
  const { name, email, errors } = state.context;

  return (
    <form onSubmit={e => { e.preventDefault(); send('SUBMIT'); }}>
      <input
        value={name}
        onChange={e => send({ type: 'UPDATE_NAME', value: e.target.value })}
      />
      {errors.name && <span>{errors.name}</span>}

      <input
        value={email}
        onChange={e => send({ type: 'UPDATE_EMAIL', value: e.target.value })}
      />
      {errors.email && <span>{errors.email}</span>}

      <button type="submit" disabled={state.matches('submitting')}>
        {state.matches('submitting') ? 'Submitting...' : 'Submit'}
      </button>

      {state.matches('success') && <p>Success!</p>}
    </form>
  );
}
```

---

## Real-World: Multi-Step Wizard

```typescript
const wizardMachine = createMachine({
  id: "wizard",
  initial: "personal",
  context: {
    personalInfo: {},
    addressInfo: {},
    paymentInfo: {},
  },
  states: {
    personal: {
      on: {
        NEXT: {
          target: "address",
          cond: (context) => !!context.personalInfo.name,
        },
      },
    },
    address: {
      on: {
        NEXT: {
          target: "payment",
          cond: (context) => !!context.addressInfo.street,
        },
        BACK: "personal",
      },
    },
    payment: {
      on: {
        BACK: "address",
        SUBMIT: "submitting",
      },
    },
    submitting: {
      invoke: {
        src: "submitWizard",
        onDone: "complete",
        onError: "payment",
      },
    },
    complete: {
      type: "final",
    },
  },
});
```

---

## TypeScript Integration

```typescript
import { createMachine, assign, State } from "xstate";

// Define types
interface ToggleContext {
  count: number;
}

type ToggleEvent = { type: "TOGGLE" } | { type: "INCREMENT" };

type ToggleState =
  | { value: "inactive"; context: ToggleContext }
  | { value: "active"; context: ToggleContext };

// Fully typed machine
const toggleMachine = createMachine<ToggleContext, ToggleEvent, ToggleState>({
  id: "toggle",
  initial: "inactive",
  context: {
    count: 0,
  },
  states: {
    inactive: {
      on: {
        TOGGLE: "active",
        INCREMENT: {
          actions: assign({
            count: (ctx) => ctx.count + 1,
          }),
        },
      },
    },
    active: {
      on: {
        TOGGLE: "inactive",
      },
    },
  },
});

// Type-safe usage
const [state, send] = useMachine(toggleMachine);
state.context.count; // number
send({ type: "TOGGLE" }); // ✅
send({ type: "INVALID" }); // ❌ Type error
```

---

## Best Practices

✅ **Model all possible states explicitly**  
✅ **Use guards for conditional transitions**  
✅ **Leverage hierarchical states** for complex UIs  
✅ **Test state machines in isolation**  
✅ **Use TypeScript for type safety**  
✅ **Visualize state machines** with XState Viz  
❌ **Don't put all state in machines** - use local state  
❌ **Don't create overly complex machines**  
❌ **Don't forget to handle all events**

---

## Key Takeaways

1. **State machines prevent impossible states**
2. **All transitions are explicit** and predictable
3. **Great for complex workflows** (auth, forms, wizards)
4. **Testable in isolation** from UI
5. **TypeScript provides safety**
6. **Hierarchical states reduce complexity**
7. **Visualize and debug** with XState tools
