# 🎣 Custom Hooks Deep Dive

> Custom hooks let you extract reusable logic from components, promoting code reuse and separation of concerns. Master custom hooks to write cleaner, more maintainable React code.

---

## 📖 1. Concept Explanation

### What are Custom Hooks?

**JavaScript functions that:**

1. **Start with `use`** (naming convention)
2. **Can call other hooks** (useState, useEffect, etc.)
3. **Extract reusable logic** from components
4. **Share stateful logic** without render props or HOCs

```typescript
// Custom hook
function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);

  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  const reset = () => setCount(initialValue);

  return { count, increment, decrement, reset };
}

// Usage
function Counter() {
  const { count, increment, decrement, reset } = useCounter(10);

  return (
    <>
      <p>Count: {count}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>Reset</button>
    </>
  );
}
```

---

## 🧠 2. Why It Matters

### Without Custom Hooks

```typescript
// ❌ Duplicated logic across components
function UserProfile() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch('/api/user')
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <Spinner />;
  if (error) return <Error error={error} />;
  return <div>{data.name}</div>;
}

function Posts() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch('/api/posts')
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);

  // Same pattern duplicated!
}
```

---

### With Custom Hooks

```typescript
// ✅ Extract reusable logic
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);

  return { data, loading, error };
}

// Usage (DRY!)
function UserProfile() {
  const { data, loading, error } = useFetch<User>('/api/user');

  if (loading) return <Spinner />;
  if (error) return <Error error={error} />;
  return <div>{data.name}</div>;
}

function Posts() {
  const { data, loading, error } = useFetch<Post[]>('/api/posts');
  // Same hook, different data!
}
```

---

## ⚙️ 3. Common Custom Hook Patterns

### 1. useFetch (Data Fetching)

```typescript
interface UseFetchOptions {
  method?: 'GET' | 'POST' | 'PUT' | 'DELETE';
  headers?: Record<string, string>;
  body?: any;
}

interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

function useFetch<T>(url: string, options?: UseFetchOptions): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  const [refetchIndex, setRefetchIndex] = useState(0);

  useEffect(() => {
    const abortController = new AbortController();

    setLoading(true);
    setError(null);

    fetch(url, {
      method: options?.method || 'GET',
      headers: {
        'Content-Type': 'application/json',
        ...options?.headers
      },
      body: options?.body ? JSON.stringify(options.body) : undefined,
      signal: abortController.signal
    })
      .then(res => {
        if (!res.ok) {
          throw new Error(`HTTP ${res.status}: ${res.statusText}`);
        }
        return res.json();
      })
      .then(setData)
      .catch(err => {
        if (err.name !== 'AbortError') {
          setError(err);
        }
      })
      .finally(() => setLoading(false));

    return () => abortController.abort();
  }, [url, options?.method, refetchIndex]);

  const refetch = useCallback(() => {
    setRefetchIndex(prev => prev + 1);
  }, []);

  return { data, loading, error, refetch };
}

// Usage
function UserList() {
  const { data: users, loading, error, refetch } = useFetch<User[]>('/api/users');

  if (loading) return <Spinner />;
  if (error) return <Error error={error} onRetry={refetch} />;

  return (
    <>
      <button onClick={refetch}>Refresh</button>
      <ul>
        {users?.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </>
  );
}
```

---

### 2. useLocalStorage (Persistent State)

```typescript
function useLocalStorage<T>(key: string, initialValue: T) {
  // State to store value
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(error);
      return initialValue;
    }
  });

  // Return wrapped version of useState's setter function
  const setValue = useCallback((value: T | ((val: T) => T)) => {
    try {
      // Allow value to be a function (same API as useState)
      const valueToStore = value instanceof Function ? value(storedValue) : value;

      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(error);
    }
  }, [key, storedValue]);

  return [storedValue, setValue] as const;
}

// Usage
function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage<'light' | 'dark'>('theme', 'light');

  return (
    <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
      Current: {theme}
    </button>
  );
}
```

---

### 3. useDebounce (Debounced Value)

```typescript
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Usage with search
function SearchInput() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 500);

  const { data: results } = useFetch(`/api/search?q=${debouncedQuery}`);

  return (
    <>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <SearchResults results={results} />
    </>
  );
}
```

---

### 4. useMediaQuery (Responsive Hooks)

```typescript
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() => {
    return window.matchMedia(query).matches;
  });

  useEffect(() => {
    const mediaQuery = window.matchMedia(query);

    const handler = (event: MediaQueryListEvent) => {
      setMatches(event.matches);
    };

    // Modern API
    mediaQuery.addEventListener('change', handler);

    return () => mediaQuery.removeEventListener('change', handler);
  }, [query]);

  return matches;
}

// Usage
function ResponsiveComponent() {
  const isMobile = useMediaQuery('(max-width: 768px)');
  const isTablet = useMediaQuery('(min-width: 769px) and (max-width: 1024px)');
  const isDesktop = useMediaQuery('(min-width: 1025px)');

  return (
    <div>
      {isMobile && <MobileView />}
      {isTablet && <TabletView />}
      {isDesktop && <DesktopView />}
    </div>
  );
}
```

---

### 5. useOnClickOutside (Click Outside Detection)

```typescript
function useOnClickOutside<T extends HTMLElement>(
  ref: RefObject<T>,
  handler: (event: MouseEvent | TouchEvent) => void
) {
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      // Do nothing if clicking ref's element or descendent elements
      if (!ref.current || ref.current.contains(event.target as Node)) {
        return;
      }

      handler(event);
    };

    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);

    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref, handler]);
}

// Usage
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);
  const dropdownRef = useRef<HTMLDivElement>(null);

  useOnClickOutside(dropdownRef, () => setIsOpen(false));

  return (
    <div ref={dropdownRef}>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && (
        <div className="menu">
          <MenuItem>Option 1</MenuItem>
          <MenuItem>Option 2</MenuItem>
        </div>
      )}
    </div>
  );
}
```

---

### 6. useWindowSize (Window Dimensions)

```typescript
interface WindowSize {
  width: number;
  height: number;
}

function useWindowSize(): WindowSize {
  const [windowSize, setWindowSize] = useState<WindowSize>({
    width: window.innerWidth,
    height: window.innerHeight
  });

  useEffect(() => {
    const handleResize = () => {
      setWindowSize({
        width: window.innerWidth,
        height: window.innerHeight
      });
    };

    window.addEventListener('resize', handleResize);

    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return windowSize;
}

// Usage
function ResponsiveGrid() {
  const { width } = useWindowSize();

  const columns = width < 768 ? 1 : width < 1024 ? 2 : 3;

  return (
    <div style={{ display: 'grid', gridTemplateColumns: `repeat(${columns}, 1fr)` }}>
      {items.map(item => (
        <GridItem key={item.id}>{item.content}</GridItem>
      ))}
    </div>
  );
}
```

---

### 7. useAsync (Async Operation State)

```typescript
interface UseAsyncState<T> {
  loading: boolean;
  error: Error | null;
  data: T | null;
}

function useAsync<T>(
  asyncFunction: () => Promise<T>,
  dependencies: any[] = []
) {
  const [state, setState] = useState<UseAsyncState<T>>({
    loading: true,
    error: null,
    data: null
  });

  useEffect(() => {
    let cancelled = false;

    setState({ loading: true, error: null, data: null });

    asyncFunction()
      .then(data => {
        if (!cancelled) {
          setState({ loading: false, error: null, data });
        }
      })
      .catch(error => {
        if (!cancelled) {
          setState({ loading: false, error, data: null });
        }
      });

    return () => {
      cancelled = true;
    };
  }, dependencies);

  return state;
}

// Usage
function UserProfile({ userId }: { userId: number }) {
  const { data: user, loading, error } = useAsync(
    () => fetch(`/api/users/${userId}`).then(res => res.json()),
    [userId]
  );

  if (loading) return <Spinner />;
  if (error) return <Error error={error} />;
  return <div>{user?.name}</div>;
}
```

---

## ✅ 4. Best Practices

### 1. Naming Convention

```typescript
// ✅ DO: Start with "use"
function useCounter() {}
function useFetch() {}
function useLocalStorage() {}

// ❌ DON'T: Missing "use" prefix
function counter() {} // Not a hook!
function fetchData() {} // Not a hook!
```

---

### 2. Return Values

```typescript
// ✅ Single value: Return directly
function useCounter() {
  const [count, setCount] = useState(0);
  return count;
}

// ✅ Multiple values (order matters): Return array
function useState(initial) {
  return [state, setState];
}

// ✅ Multiple values (order doesn't matter): Return object
function useAuth() {
  return { user, login, logout, isAuthenticated };
}

// ✅ Complex return: Combine approaches
function useForm() {
  return {
    values,
    errors,
    touched,
    handleChange,
    handleSubmit,
    isValid,
    // Helper functions
    setFieldValue,
    setFieldError,
  };
}
```

---

### 3. Memoization

```typescript
// ❌ BAD: New object every render
function useFormattedDate(date: Date) {
  return {
    formatted: date.toLocaleDateString(),
    iso: date.toISOString(),
  }; // New object reference!
}

// ✅ GOOD: Memoized
function useFormattedDate(date: Date) {
  return useMemo(
    () => ({
      formatted: date.toLocaleDateString(),
      iso: date.toISOString(),
    }),
    [date],
  );
}

// ✅ GOOD: Memoized callbacks
function useCounter() {
  const [count, setCount] = useState(0);

  const increment = useCallback(() => {
    setCount((c) => c + 1);
  }, []);

  const decrement = useCallback(() => {
    setCount((c) => c - 1);
  }, []);

  return { count, increment, decrement };
}
```

---

### 4. Dependency Arrays

```typescript
// ❌ BAD: Missing dependencies
function useFetch(url: string) {
  useEffect(() => {
    fetch(url).then(/* ... */);
  }, []); // Missing url!
}

// ✅ GOOD: All dependencies included
function useFetch(url: string) {
  useEffect(() => {
    fetch(url).then(/* ... */);
  }, [url]);
}

// ✅ GOOD: No external dependencies
function useInterval(callback: () => void, delay: number) {
  const savedCallback = useRef(callback);

  useEffect(() => {
    savedCallback.current = callback;
  });

  useEffect(() => {
    const tick = () => savedCallback.current();
    const id = setInterval(tick, delay);
    return () => clearInterval(id);
  }, [delay]); // Only delay, not callback!
}
```

---

## 🧪 5. Real-World Examples

### useForm (Form Management)

```typescript
interface UseFormOptions<T> {
  initialValues: T;
  validate?: (values: T) => Partial<Record<keyof T, string>>;
  onSubmit: (values: T) => void | Promise<void>;
}

function useForm<T extends Record<string, any>>({
  initialValues,
  validate,
  onSubmit
}: UseFormOptions<T>) {
  const [values, setValues] = useState<T>(initialValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});
  const [touched, setTouched] = useState<Partial<Record<keyof T, boolean>>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleChange = useCallback((field: keyof T) => {
    return (event: React.ChangeEvent<HTMLInputElement>) => {
      setValues(prev => ({
        ...prev,
        [field]: event.target.value
      }));
    };
  }, []);

  const handleBlur = useCallback((field: keyof T) => {
    return () => {
      setTouched(prev => ({
        ...prev,
        [field]: true
      }));
    };
  }, []);

  const handleSubmit = useCallback(async (event: React.FormEvent) => {
    event.preventDefault();

    // Mark all fields as touched
    setTouched(
      Object.keys(values).reduce((acc, key) => ({
        ...acc,
        [key]: true
      }), {})
    );

    // Validate
    const validationErrors = validate?.(values) || {};
    setErrors(validationErrors);

    if (Object.keys(validationErrors).length === 0) {
      setIsSubmitting(true);
      try {
        await onSubmit(values);
      } finally {
        setIsSubmitting(false);
      }
    }
  }, [values, validate, onSubmit]);

  const reset = useCallback(() => {
    setValues(initialValues);
    setErrors({});
    setTouched({});
  }, [initialValues]);

  return {
    values,
    errors,
    touched,
    isSubmitting,
    handleChange,
    handleBlur,
    handleSubmit,
    reset
  };
}

// Usage
interface LoginForm {
  email: string;
  password: string;
}

function LoginPage() {
  const form = useForm<LoginForm>({
    initialValues: {
      email: '',
      password: ''
    },
    validate: (values) => {
      const errors: Partial<Record<keyof LoginForm, string>> = {};

      if (!values.email) {
        errors.email = 'Email required';
      } else if (!/\S+@\S+\.\S+/.test(values.email)) {
        errors.email = 'Invalid email';
      }

      if (!values.password) {
        errors.password = 'Password required';
      } else if (values.password.length < 8) {
        errors.password = 'Password must be 8+ characters';
      }

      return errors;
    },
    onSubmit: async (values) => {
      await fetch('/api/login', {
        method: 'POST',
        body: JSON.stringify(values)
      });
    }
  });

  return (
    <form onSubmit={form.handleSubmit}>
      <input
        type="email"
        value={form.values.email}
        onChange={form.handleChange('email')}
        onBlur={form.handleBlur('email')}
      />
      {form.touched.email && form.errors.email && (
        <span>{form.errors.email}</span>
      )}

      <input
        type="password"
        value={form.values.password}
        onChange={form.handleChange('password')}
        onBlur={form.handleBlur('password')}
      />
      {form.touched.password && form.errors.password && (
        <span>{form.errors.password}</span>
      )}

      <button type="submit" disabled={form.isSubmitting}>
        {form.isSubmitting ? 'Logging in...' : 'Login'}
      </button>
    </form>
  );
}
```

---

## ❓ 6. Interview Questions

### Q1: What makes a function a "hook"?

**Answer:**

**Requirements:**

1. **Name starts with `use`** (convention)
2. **Can call other hooks** (useState, useEffect, etc.)
3. **Must be called at top level** (not in loops/conditions)
4. **Must be called in React function components or custom hooks**

```typescript
// ✅ Valid hook
function useCounter() {
  const [count, setCount] = useState(0);
  return count;
}

// ❌ Not a hook (missing "use" prefix)
function counter() {
  const [count, setCount] = useState(0);
  return count;
}

// ❌ Not a hook (regular function, can't call hooks)
function formatDate(date: Date) {
  // Can't call useState here!
  return date.toLocaleDateString();
}
```

---

### Q2: When should you create a custom hook?

**Answer:**

**Create custom hooks when:**

1. **Duplicated logic across components:**

```typescript
// Multiple components doing same fetch logic → useFetch hook
```

2. **Complex logic needs encapsulation:**

```typescript
// Complex form validation → useForm hook
// WebSocket connection management → useWebSocket hook
```

3. **Combining multiple hooks:**

```typescript
// useState + useEffect + useCallback → useAsync hook
```

**Don't create custom hooks for:**

- One-time logic (just write inline)
- Simple calculations (regular functions)
- Component-specific logic (keep in component)

---

### Q3: How to test custom hooks?

**Answer:**

**Use `@testing-library/react-hooks`:**

```typescript
import { renderHook, act } from "@testing-library/react-hooks";
import { useCounter } from "./useCounter";

describe("useCounter", () => {
  it("should increment counter", () => {
    const { result } = renderHook(() => useCounter(0));

    expect(result.current.count).toBe(0);

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it("should reset counter", () => {
    const { result } = renderHook(() => useCounter(10));

    act(() => {
      result.current.increment();
      result.current.reset();
    });

    expect(result.current.count).toBe(10);
  });
});
```

---

## 🎯 Summary

**Custom hooks:**

- Extract reusable logic
- Start with `use` prefix
- Can call other hooks
- Return state and functions

**Common patterns:**

- useFetch (data fetching)
- useLocalStorage (persistent state)
- useDebounce (debounced values)
- useMediaQuery (responsive)
- useOnClickOutside (click detection)

**Best practices:**

- ✅ Name with `use` prefix
- ✅ Return object for multiple values
- ✅ Memoize callbacks and objects
- ✅ Include all dependencies
- ✅ Test hooks in isolation

Master custom hooks for reusable, maintainable React code! 🎣
