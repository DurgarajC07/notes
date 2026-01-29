# 🎣 Hooks Typing in TypeScript

> **Type-Safe React Hooks - useState, useRef, useEffect, Custom Hooks**

---

## 📋 Table of Contents

- [useState Typing](#usestate-typing)
- [useRef Typing](#useref-typing)
- [useEffect & useLayoutEffect](#useeffect--uselayouteffect)
- [useContext Typing](#usecontext-typing)
- [useReducer Typing](#usereducer-typing)
- [useMemo & useCallback](#usememo--usecallback)
- [Custom Hooks](#custom-hooks)
- [Advanced Patterns](#advanced-patterns)
- [Common Pitfalls](#common-pitfalls)
- [Best Practices](#best-practices)

---

## useState Typing

### **Basic Usage**

```typescript
import { useState } from "react";

// ✅ Type inferred from initial value
const [count, setCount] = useState(0); // number
const [name, setName] = useState(""); // string
const [isOpen, setOpen] = useState(false); // boolean

// ✅ Explicit type annotation
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<Item[]>([]);
```

### **With Unions**

```typescript
type Status = "idle" | "loading" | "success" | "error";

function Component() {
  // ✅ Type narrowing works
  const [status, setStatus] = useState<Status>("idle");

  setStatus("loading"); // ✅ Works
  setStatus("invalid"); // ❌ Type error

  if (status === "loading") {
    // TypeScript knows status is 'loading'
  }
}
```

### **With Complex Objects**

```typescript
interface FormState {
  name: string;
  email: string;
  age: number;
}

function Form() {
  const [form, setForm] = useState<FormState>({
    name: "",
    email: "",
    age: 0,
  });

  // ✅ Type-safe updates
  setForm((prev) => ({
    ...prev,
    name: "John", // ✅ Works
    invalid: "field", // ❌ Type error
  }));
}
```

### **Lazy Initialization**

```typescript
// ✅ Function return type is inferred
const [data, setData] = useState(() => {
  const stored = localStorage.getItem("data");
  return stored ? JSON.parse(stored) : [];
});

// ✅ With type annotation
const [config, setConfig] = useState<Config>(() => {
  return loadConfig();
});

// ✅ Async initialization pattern
const [user, setUser] = useState<User | null>(null);

useEffect(() => {
  loadUser().then(setUser);
}, []);
```

### **Functional Updates**

```typescript
const [count, setCount] = useState(0);

// ✅ Callback receives previous value
setCount((prev) => prev + 1); // prev is number

const [users, setUsers] = useState<User[]>([]);

// ✅ Type-safe array operations
setUsers((prev) => [...prev, newUser]);
setUsers((prev) => prev.filter((u) => u.id !== deletedId));
setUsers((prev) => prev.map((u) => (u.id === id ? updatedUser : u)));
```

---

## useRef Typing

### **DOM References**

```typescript
import { useRef } from 'react';

function Component() {
  // ✅ Ref for div element
  const divRef = useRef<HTMLDivElement>(null);

  // ✅ Ref for input element
  const inputRef = useRef<HTMLInputElement>(null);

  // ✅ Ref for button element
  const buttonRef = useRef<HTMLButtonElement>(null);

  useEffect(() => {
    // TypeScript knows these might be null
    if (divRef.current) {
      divRef.current.scrollIntoView();
    }

    if (inputRef.current) {
      inputRef.current.focus();
    }
  }, []);

  return (
    <>
      <div ref={divRef} />
      <input ref={inputRef} />
      <button ref={buttonRef} />
    </>
  );
}
```

### **Mutable Values**

```typescript
// ✅ Mutable ref (doesn't need initial null)
const countRef = useRef(0);
const timerRef = useRef<number>();
const previousValueRef = useRef<string>();

useEffect(() => {
  // ✅ Can mutate directly
  countRef.current += 1;

  timerRef.current = window.setTimeout(() => {
    // ...
  }, 1000);

  return () => {
    if (timerRef.current) {
      clearTimeout(timerRef.current);
    }
  };
}, []);
```

### **Ref vs MutableRefObject**

```typescript
// Ref<T> - for DOM elements (can be null)
const domRef = useRef<HTMLDivElement>(null);
domRef.current = document.createElement("div"); // ❌ Error: readonly

// MutableRefObject<T> - for mutable values
const mutableRef = useRef(0);
mutableRef.current = 10; // ✅ Works

// ✅ Explicit MutableRefObject
const explicit = useRef<number | undefined>(undefined);
// Type: MutableRefObject<number | undefined>
```

### **useImperativeHandle**

```typescript
import { useImperativeHandle, forwardRef } from 'react';

// Define handle type
interface InputHandle {
  focus: () => void;
  getValue: () => string;
  clear: () => void;
}

// ✅ Forward ref with handle type
const Input = forwardRef<InputHandle, { placeholder?: string }>((props, ref) => {
  const inputRef = useRef<HTMLInputElement>(null);

  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current?.focus();
    },
    getValue: () => {
      return inputRef.current?.value ?? '';
    },
    clear: () => {
      if (inputRef.current) {
        inputRef.current.value = '';
      }
    },
  }));

  return <input ref={inputRef} placeholder={props.placeholder} />;
});

// Usage
function Parent() {
  const inputRef = useRef<InputHandle>(null);

  return (
    <>
      <Input ref={inputRef} />
      <button onClick={() => inputRef.current?.focus()}>
        Focus
      </button>
    </>
  );
}
```

---

## useEffect & useLayoutEffect

### **Basic Typing**

```typescript
import { useEffect, useLayoutEffect } from "react";

function Component() {
  // ✅ No return value
  useEffect(() => {
    console.log("Effect ran");
  }, []);

  // ✅ Return cleanup function
  useEffect(() => {
    const timer = setTimeout(() => {}, 1000);

    return () => {
      clearTimeout(timer);
    };
  }, []);

  // ❌ Can't return Promise
  useEffect(async () => {
    // Error
    await fetchData();
  }, []);

  // ✅ Async inside effect
  useEffect(() => {
    async function load() {
      await fetchData();
    }
    load();
  }, []);
}
```

### **Cleanup Function Type**

```typescript
// Cleanup must return void or undefined
useEffect(() => {
  const subscription = api.subscribe();

  return () => {
    subscription.unsubscribe();
  }; // ✅ Returns void
}, []);

// ❌ Can't return anything else
useEffect(() => {
  return 123; // Error: Type 'number' is not assignable to void
}, []);
```

### **Dependency Array Typing**

```typescript
function Component({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);

  // ✅ Dependencies are checked
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]); // Must include userId

  // ⚠️ ESLint will warn about missing dependencies
  useEffect(() => {
    console.log(user);
  }, []); // Should include 'user'
}
```

---

## useContext Typing

### **Basic Context**

```typescript
import { createContext, useContext } from 'react';

// ✅ Define context type
interface ThemeContext {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

// ✅ Create context with type
const ThemeContext = createContext<ThemeContext | undefined>(undefined);

// ✅ Type-safe hook
function useTheme() {
  const context = useContext(ThemeContext);

  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }

  return context;
}

// Provider
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  };

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// Usage
function Component() {
  const { theme, toggleTheme } = useTheme(); // Type-safe!

  return (
    <button onClick={toggleTheme}>
      Current theme: {theme}
    </button>
  );
}
```

### **Context with Default Value**

```typescript
// ✅ With default value (no undefined)
const ThemeContext = createContext<ThemeContext>({
  theme: "light",
  toggleTheme: () => {},
});

// ✅ Simpler hook (no null check needed)
function useTheme() {
  return useContext(ThemeContext);
}
```

### **Generic Context**

```typescript
function createSafeContext<T>() {
  const Context = createContext<T | undefined>(undefined);

  function useCtx() {
    const context = useContext(Context);
    if (!context) {
      throw new Error('Context must be used within Provider');
    }
    return context;
  }

  return [Context.Provider, useCtx] as const;
}

// Usage
interface UserContext {
  user: User;
  logout: () => void;
}

const [UserProvider, useUser] = createSafeContext<UserContext>();

function App() {
  const [user, setUser] = useState<User>(currentUser);

  return (
    <UserProvider value={{ user, logout: () => setUser(null) }}>
      <Dashboard />
    </UserProvider>
  );
}

function Dashboard() {
  const { user, logout } = useUser(); // Type-safe, no null check!
  return <div>{user.name}</div>;
}
```

---

## useReducer Typing

### **Basic Reducer**

```typescript
import { useReducer } from 'react';

// ✅ Define state and actions
type State = {
  count: number;
  error: string | null;
};

type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'reset' }
  | { type: 'error'; error: string };

// ✅ Type-safe reducer
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + 1 };

    case 'decrement':
      return { ...state, count: state.count - 1 };

    case 'reset':
      return { count: 0, error: null };

    case 'error':
      return { ...state, error: action.error };

    default:
      // ✅ Exhaustiveness check
      const _: never = action;
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, {
    count: 0,
    error: null,
  });

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </div>
  );
}
```

### **With Payload**

```typescript
type State = {
  users: User[];
  loading: boolean;
};

type Action =
  | { type: "FETCH_START" }
  | { type: "FETCH_SUCCESS"; payload: User[] }
  | { type: "FETCH_ERROR"; payload: Error }
  | { type: "ADD_USER"; payload: User }
  | { type: "DELETE_USER"; payload: string };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "FETCH_START":
      return { ...state, loading: true };

    case "FETCH_SUCCESS":
      return {
        users: action.payload, // ✅ Payload is User[]
        loading: false,
      };

    case "ADD_USER":
      return {
        ...state,
        users: [...state.users, action.payload],
      };

    case "DELETE_USER":
      return {
        ...state,
        users: state.users.filter((u) => u.id !== action.payload),
      };

    default:
      return state;
  }
}
```

---

## useMemo & useCallback

### **useMemo Typing**

```typescript
import { useMemo } from "react";

function Component({ items }: { items: Item[] }) {
  // ✅ Type inferred from return value
  const sortedItems = useMemo(() => {
    return [...items].sort((a, b) => a.name.localeCompare(b.name));
  }, [items]);
  // Type: Item[]

  // ✅ Explicit type annotation
  const expensiveValue = useMemo<number>(() => {
    return items.reduce((sum, item) => sum + item.price, 0);
  }, [items]);

  // ✅ Complex computations
  const stats = useMemo(() => {
    return {
      count: items.length,
      total: items.reduce((sum, item) => sum + item.price, 0),
      average:
        items.length > 0
          ? items.reduce((sum, item) => sum + item.price, 0) / items.length
          : 0,
    };
  }, [items]);
  // Type: { count: number; total: number; average: number }
}
```

### **useCallback Typing**

```typescript
import { useCallback } from "react";

function Component() {
  // ✅ Type inferred from function signature
  const handleClick = useCallback(() => {
    console.log("clicked");
  }, []);
  // Type: () => void

  // ✅ With parameters
  const handleChange = useCallback((value: string) => {
    console.log(value);
  }, []);
  // Type: (value: string) => void

  // ✅ With dependencies
  const [multiplier, setMultiplier] = useState(2);

  const calculate = useCallback(
    (value: number) => {
      return value * multiplier;
    },
    [multiplier],
  );
  // Type: (value: number) => number
}
```

### **Generic useCallback**

```typescript
function useEvent<T extends (...args: any[]) => any>(callback: T): T {
  const ref = useRef(callback);

  useLayoutEffect(() => {
    ref.current = callback;
  }, [callback]);

  return useCallback(
    ((...args) => {
      return ref.current(...args);
    }) as T,
    [],
  );
}

// Usage - callback never changes reference
const handleSubmit = useEvent((data: FormData) => {
  console.log(data);
});
```

---

## Custom Hooks

### **Basic Custom Hook**

```typescript
// ✅ Return type inferred
function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);

  const increment = () => setCount((c) => c + 1);
  const decrement = () => setCount((c) => c - 1);
  const reset = () => setCount(initialValue);

  return { count, increment, decrement, reset };
}

// Type: {
//   count: number;
//   increment: () => void;
//   decrement: () => void;
//   reset: () => void;
// }
```

### **With Explicit Return Type**

```typescript
interface UseToggleReturn {
  value: boolean;
  toggle: () => void;
  setTrue: () => void;
  setFalse: () => void;
}

function useToggle(initial = false): UseToggleReturn {
  const [value, setValue] = useState(initial);

  return {
    value,
    toggle: () => setValue((v) => !v),
    setTrue: () => setValue(true),
    setFalse: () => setValue(false),
  };
}
```

### **Generic Custom Hook**

```typescript
function useArray<T>(initialValue: T[] = []) {
  const [array, setArray] = useState<T[]>(initialValue);

  return {
    array,
    push: (item: T) => setArray((prev) => [...prev, item]),
    filter: (callback: (item: T) => boolean) =>
      setArray((prev) => prev.filter(callback)),
    update: (index: number, item: T) =>
      setArray((prev) => [
        ...prev.slice(0, index),
        item,
        ...prev.slice(index + 1),
      ]),
    remove: (index: number) =>
      setArray((prev) => [...prev.slice(0, index), ...prev.slice(index + 1)]),
    clear: () => setArray([]),
  };
}

// Usage
const numbers = useArray<number>([1, 2, 3]);
numbers.push(4); // ✅ Type-safe
numbers.push("5"); // ❌ Type error

const users = useArray<User>([]);
users.push({ id: "1", name: "John" }); // ✅ Works
```

### **Hook with Options**

```typescript
interface UseFetchOptions {
  method?: 'GET' | 'POST' | 'PUT' | 'DELETE';
  headers?: Record<string, string>;
  body?: any;
}

interface UseFetchReturn<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => Promise<void>;
}

function useFetch<T>(
  url: string,
  options?: UseFetchOptions
): UseFetchReturn<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const fetchData = useCallback(async () => {
    try {
      setLoading(true);
      const response = await fetch(url, options);
      const json = await response.json();
      setData(json);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
    }
  }, [url, options]);

  useEffect(() => {
    fetchData();
  }, [fetchData]);

  return { data, loading, error, refetch: fetchData };
}

// Usage
interface User {
  id: string;
  name: string;
}

function Component() {
  const { data, loading, error } = useFetch<User>('/api/user');

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!data) return <div>No data</div>;

  return <div>{data.name}</div>; // ✅ Type-safe
}
```

---

## Advanced Patterns

### **Conditional Hook Return**

```typescript
function useData<T>(shouldFetch: boolean): T | null {
  const [data, setData] = useState<T | null>(null);

  useEffect(() => {
    if (shouldFetch) {
      fetchData<T>().then(setData);
    }
  }, [shouldFetch]);

  return data;
}
```

### **Hook with Overloads**

```typescript
function useState<T>(): [
  T | undefined,
  Dispatch<SetStateAction<T | undefined>>,
];
function useState<T>(initialValue: T): [T, Dispatch<SetStateAction<T>>];
function useState<T = undefined>() {
  // Implementation
}

// Similar pattern for custom hooks
function useLocalStorage(key: string): [string | null, (value: string) => void];
function useLocalStorage<T>(
  key: string,
  parse: true,
): [T | null, (value: T) => void];
function useLocalStorage<T = string>(key: string, parse = false) {
  // Implementation
}
```

### **Dependent Hooks**

```typescript
function useAuthenticatedFetch<T>(url: string) {
  const { token } = useAuth(); // Must call hooks unconditionally
  const result = useFetch<T>(url, {
    headers: token ? { Authorization: `Bearer ${token}` } : undefined,
  });

  return result;
}
```

---

## Common Pitfalls

### **1. useRef initial value**

```typescript
// ❌ Bad: Ref might be null
const divRef = useRef<HTMLDivElement>();
divRef.current.scrollIntoView(); // Error: might be undefined

// ✅ Good: Explicitly handle null
const divRef = useRef<HTMLDivElement>(null);
if (divRef.current) {
  divRef.current.scrollIntoView();
}
```

### **2. useState with null/undefined**

```typescript
// ❌ Bad: Type is User, can't be null
const [user, setUser] = useState<User>({} as User);

// ✅ Good: Explicitly include null
const [user, setUser] = useState<User | null>(null);
```

### **3. useEffect async**

```typescript
// ❌ Bad: Can't use async directly
useEffect(async () => {
  await fetchData();
}, []);

// ✅ Good: Wrap in async function
useEffect(() => {
  async function load() {
    await fetchData();
  }
  load();
}, []);

// ✅ Good: IIFE
useEffect(() => {
  (async () => {
    await fetchData();
  })();
}, []);
```

### **4. Exhaustive deps**

```typescript
// ⚠️ ESLint warning: missing dependency
useEffect(() => {
  console.log(someValue);
}, []); // Should include someValue

// ✅ Include all dependencies
useEffect(() => {
  console.log(someValue);
}, [someValue]);

// ✅ Or use useCallback/useMemo for stable references
const stableCallback = useCallback(() => {
  // ...
}, []);

useEffect(() => {
  stableCallback();
}, [stableCallback]);
```

---

## Best Practices

### **1. Type inference over explicit types**

```typescript
// ✅ Good: Type inferred
const [count, setCount] = useState(0);

// ⚠️ Unnecessary: Explicit type
const [count, setCount] = useState<number>(0);
```

### **2. Use discriminated unions**

```typescript
type AsyncState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

function useAsync<T>() {
  const [state, setState] = useState<AsyncState<T>>({ status: "idle" });

  // Type-safe state handling
  if (state.status === "success") {
    console.log(state.data); // ✅ data exists
  }

  return state;
}
```

### **3. Extract hook types**

```typescript
const counter = useCounter();
type CounterHook = ReturnType<typeof useCounter>;
// Use CounterHook type elsewhere
```

### **4. Document generic parameters**

```typescript
/**
 * Custom hook for array state management
 * @template T The type of items in the array
 */
function useArray<T>(initialValue: T[] = []) {
  // ...
}
```

---

## Summary

### **Key Patterns**

1. **useState**: Include null/undefined in union when needed
2. **useRef**: Use `null` for DOM refs, value for mutable refs
3. **useEffect**: Never return Promise, use cleanup function
4. **useContext**: Create custom hook with null check
5. **useReducer**: Use discriminated unions for actions
6. **Custom hooks**: Generic hooks for reusability

### **Type Safety Checklist**

- [ ] All hook dependencies typed correctly
- [ ] Null/undefined handled in refs and state
- [ ] Custom hooks have proper generic constraints
- [ ] Context hooks throw on missing provider
- [ ] Effect cleanup functions return void
- [ ] Exhaustive type checking in reducers

---

**Interview Questions:**

1. How do you type useState with null values?
2. What's the difference between Ref and MutableRefObject?
3. Why can't useEffect be async?
4. How do you create type-safe context?

**Practice:**

- Build type-safe custom hooks
- Create generic hook utilities
- Implement useReducer with discriminated unions
- Type complex ref patterns
