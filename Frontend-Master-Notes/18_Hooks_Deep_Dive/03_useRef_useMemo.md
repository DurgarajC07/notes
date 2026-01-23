# 🎯 useRef & useMemo Deep Dive

> `useRef` for mutable references without re-renders, `useMemo` for expensive computation caching. Master these hooks to optimize React performance and manage side effects.

---

## 📖 1. Concept Explanation

### useRef

**Mutable container that persists across renders without triggering re-renders:**

```typescript
const ref = useRef<T>(initialValue);

// Returns: { current: initialValue }
// Mutating ref.current doesn't trigger re-render
```

**Use cases:**

1. **DOM access** (focus input, measure size)
2. **Previous value tracking** (compare prev vs current)
3. **Mutable values** (timers, subscriptions, flags)
4. **Instance variables** (class components equivalent)

---

### useMemo

**Cache expensive computations, recompute only when dependencies change:**

```typescript
const memoizedValue = useMemo(
  () => expensiveComputation(a, b),
  [a, b], // Dependencies
);
```

**Use cases:**

1. **Expensive calculations** (filtering, sorting, transforming large datasets)
2. **Referential equality** (prevent unnecessary child re-renders)
3. **Derived state** (computed values from props/state)

---

## 🧠 2. Why It Matters

### Without useRef

```typescript
// ❌ BAD: State causes unnecessary re-renders
function Timer() {
  const [intervalId, setIntervalId] = useState<number | null>(null);

  useEffect(() => {
    const id = setInterval(() => {
      console.log("Tick");
    }, 1000);

    setIntervalId(id); // Triggers re-render!

    return () => clearInterval(id);
  }, []);
}
```

```typescript
// ✅ GOOD: Ref doesn't trigger re-renders
function Timer() {
  const intervalIdRef = useRef<number | null>(null);

  useEffect(() => {
    intervalIdRef.current = setInterval(() => {
      console.log("Tick");
    }, 1000);

    return () => {
      if (intervalIdRef.current) {
        clearInterval(intervalIdRef.current);
      }
    };
  }, []);
}
```

---

### Without useMemo

```typescript
// ❌ BAD: Re-computes on every render
function ExpensiveList({ items, filter }) {
  // Every render, even if items/filter unchanged!
  const filteredItems = items.filter(item =>
    item.name.toLowerCase().includes(filter.toLowerCase())
  );

  return <List items={filteredItems} />;
}
```

```typescript
// ✅ GOOD: Memoized, recomputes only when dependencies change
function ExpensiveList({ items, filter }) {
  const filteredItems = useMemo(
    () => items.filter(item =>
      item.name.toLowerCase().includes(filter.toLowerCase())
    ),
    [items, filter]
  );

  return <List items={filteredItems} />;
}
```

---

## ⚙️ 3. useRef Deep Dive

### 1. DOM Access

```typescript
import { useRef, useEffect } from 'react';

function SearchInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    // Auto-focus on mount
    inputRef.current?.focus();
  }, []);

  const handleClear = () => {
    if (inputRef.current) {
      inputRef.current.value = '';
      inputRef.current.focus();
    }
  };

  return (
    <>
      <input ref={inputRef} type="text" />
      <button onClick={handleClear}>Clear</button>
    </>
  );
}
```

---

### 2. Previous Value Tracking

```typescript
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();

  useEffect(() => {
    ref.current = value; // Update after render
  });

  return ref.current; // Return previous value
}

// Usage
function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);

  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {prevCount ?? 'N/A'}</p>
      {prevCount !== undefined && (
        <p>Changed by: {count - prevCount}</p>
      )}
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
}
```

**Execution order:**

1. Render with `count = 5`
2. `usePrevious` returns `ref.current = 4` (previous value)
3. After render, `useEffect` runs → `ref.current = 5`
4. Next render, `usePrevious` returns `5`

---

### 3. Mutable Values (No Re-render)

```typescript
function VideoPlayer({ src }: { src: string }) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const isPlayingRef = useRef(false);

  const togglePlay = () => {
    const video = videoRef.current;
    if (!video) return;

    if (isPlayingRef.current) {
      video.pause();
      isPlayingRef.current = false;
    } else {
      video.play();
      isPlayingRef.current = true;
    }

    // No re-render triggered!
  };

  return (
    <>
      <video ref={videoRef} src={src} />
      <button onClick={togglePlay}>
        {isPlayingRef.current ? 'Pause' : 'Play'}
      </button>
    </>
  );
}
```

---

### 4. Cleanup with Refs

```typescript
function DataFetcher({ url }: { url: string }) {
  const abortControllerRef = useRef<AbortController | null>(null);

  useEffect(() => {
    // Cancel previous request
    abortControllerRef.current?.abort();

    // Create new controller
    const controller = new AbortController();
    abortControllerRef.current = controller;

    fetch(url, { signal: controller.signal })
      .then(res => res.json())
      .then(data => console.log(data))
      .catch(err => {
        if (err.name === 'AbortError') {
          console.log('Request cancelled');
        }
      });

    return () => {
      controller.abort();
    };
  }, [url]);

  return <div>Fetching data...</div>;
}
```

---

### 5. Forwarding Refs

```typescript
import { forwardRef, useRef, useImperativeHandle } from 'react';

// Child component
interface InputHandle {
  focus: () => void;
  clear: () => void;
}

const CustomInput = forwardRef<InputHandle, { placeholder?: string }>(
  (props, ref) => {
    const inputRef = useRef<HTMLInputElement>(null);

    // Expose custom methods to parent
    useImperativeHandle(ref, () => ({
      focus() {
        inputRef.current?.focus();
      },
      clear() {
        if (inputRef.current) {
          inputRef.current.value = '';
        }
      }
    }));

    return <input ref={inputRef} {...props} />;
  }
);

// Parent component
function Form() {
  const inputRef = useRef<InputHandle>(null);

  const handleSubmit = () => {
    // Access child methods
    inputRef.current?.clear();
    inputRef.current?.focus();
  };

  return (
    <>
      <CustomInput ref={inputRef} placeholder="Name" />
      <button onClick={handleSubmit}>Submit</button>
    </>
  );
}
```

---

## ⚙️ 4. useMemo Deep Dive

### 1. Expensive Computation Caching

```typescript
interface Item {
  id: number;
  name: string;
  price: number;
  category: string;
}

function ProductList({ items, filter, sortBy }: {
  items: Item[];
  filter: string;
  sortBy: 'name' | 'price';
}) {
  // WITHOUT useMemo: Runs on every render (parent state change)
  // WITH useMemo: Only runs when items/filter/sortBy change

  const processedItems = useMemo(() => {
    console.log('Processing items...');

    // 1. Filter
    let result = items.filter(item =>
      item.name.toLowerCase().includes(filter.toLowerCase())
    );

    // 2. Sort
    result = result.sort((a, b) => {
      if (sortBy === 'name') {
        return a.name.localeCompare(b.name);
      }
      return a.price - b.price;
    });

    return result;
  }, [items, filter, sortBy]);

  return (
    <ul>
      {processedItems.map(item => (
        <li key={item.id}>
          {item.name} - ${item.price}
        </li>
      ))}
    </ul>
  );
}
```

**Benchmark:**

```
Without useMemo: ~15ms per render (1000+ items)
With useMemo:    ~0.1ms (when dependencies unchanged)
```

---

### 2. Referential Equality (Prevent Child Re-renders)

```typescript
import { memo } from 'react';

// Child component (memoized)
const ExpensiveChild = memo(({ data }: { data: any[] }) => {
  console.log('ExpensiveChild rendered');
  return <div>Items: {data.length}</div>;
});

// ❌ BAD: Child re-renders every time
function Parent() {
  const [count, setCount] = useState(0);

  const data = [1, 2, 3]; // New array every render!

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>
        Count: {count}
      </button>
      <ExpensiveChild data={data} /> {/* Re-renders! */}
    </>
  );
}

// ✅ GOOD: Child only re-renders when data changes
function Parent() {
  const [count, setCount] = useState(0);

  const data = useMemo(() => [1, 2, 3], []); // Same reference!

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>
        Count: {count}
      </button>
      <ExpensiveChild data={data} /> {/* No re-render! */}
    </>
  );
}
```

---

### 3. Derived State

```typescript
function ShoppingCart({ items }: { items: CartItem[] }) {
  const subtotal = useMemo(
    () => items.reduce((sum, item) => sum + item.price * item.quantity, 0),
    [items]
  );

  const tax = useMemo(
    () => subtotal * 0.08,
    [subtotal]
  );

  const shipping = useMemo(
    () => subtotal > 50 ? 0 : 5.99,
    [subtotal]
  );

  const total = useMemo(
    () => subtotal + tax + shipping,
    [subtotal, tax, shipping]
  );

  return (
    <div>
      <p>Subtotal: ${subtotal.toFixed(2)}</p>
      <p>Tax: ${tax.toFixed(2)}</p>
      <p>Shipping: ${shipping.toFixed(2)}</p>
      <h3>Total: ${total.toFixed(2)}</h3>
    </div>
  );
}
```

---

### 4. When NOT to Use useMemo

```typescript
// ❌ Over-optimization (premature optimization)
function Counter() {
  const [count, setCount] = useState(0);

  // Unnecessary! Simple calculation is fast
  const doubled = useMemo(() => count * 2, [count]);

  return <div>{doubled}</div>;
}

// ✅ Just compute directly
function Counter() {
  const [count, setCount] = useState(0);
  const doubled = count * 2; // Fast enough!
  return <div>{doubled}</div>;
}
```

**Rule of thumb:**

- ✅ Use useMemo: >10ms computation, large arrays (1000+ items)
- ❌ Don't use useMemo: Simple math, string concatenation, small arrays

---

## ✅ 5. Best Practices

### useRef

```typescript
// ✅ DO: Store mutable values
const timerRef = useRef<number | null>(null);

// ✅ DO: Access DOM elements
const inputRef = useRef<HTMLInputElement>(null);

// ❌ DON'T: Use for state that should trigger re-renders
const countRef = useRef(0); // Use useState instead!

// ❌ DON'T: Access ref.current during render
function Bad() {
  const ref = useRef(0);
  ref.current += 1; // Side effect during render!
  return <div>{ref.current}</div>;
}

// ✅ DO: Mutate in effects or event handlers
function Good() {
  const ref = useRef(0);

  useEffect(() => {
    ref.current += 1; // Safe in effect
  });

  return <div>{ref.current}</div>;
}
```

---

### useMemo

```typescript
// ✅ DO: Memoize expensive computations
const filtered = useMemo(
  () => largeArray.filter((item) => item.active),
  [largeArray],
);

// ✅ DO: Maintain referential equality
const config = useMemo(() => ({ key: "value" }), []);

// ❌ DON'T: Memoize simple values
const sum = useMemo(() => a + b, [a, b]); // Overkill!

// ❌ DON'T: Have side effects in useMemo
const value = useMemo(() => {
  fetchData(); // Side effect! Use useEffect instead
  return computedValue;
}, []);
```

---

## 🧪 6. Real-World Examples

### Virtual Scroll with useRef

```typescript
function VirtualScroll({ items }: { items: any[] }) {
  const containerRef = useRef<HTMLDivElement>(null);
  const [visibleRange, setVisibleRange] = useState({ start: 0, end: 20 });

  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    const handleScroll = () => {
      const scrollTop = container.scrollTop;
      const itemHeight = 50;

      const start = Math.floor(scrollTop / itemHeight);
      const end = start + 20;

      setVisibleRange({ start, end });
    };

    container.addEventListener('scroll', handleScroll);
    return () => container.removeEventListener('scroll', handleScroll);
  }, []);

  const visibleItems = items.slice(visibleRange.start, visibleRange.end);

  return (
    <div ref={containerRef} style={{ height: '400px', overflow: 'auto' }}>
      {visibleItems.map((item, i) => (
        <div key={visibleRange.start + i} style={{ height: '50px' }}>
          {item}
        </div>
      ))}
    </div>
  );
}
```

---

### Search with Highlighting (useMemo)

```typescript
interface SearchResult {
  id: number;
  text: string;
}

function SearchResults({ query, results }: {
  query: string;
  results: SearchResult[];
}) {
  const highlightedResults = useMemo(() => {
    if (!query) return results;

    const regex = new RegExp(`(${query})`, 'gi');

    return results.map(result => ({
      ...result,
      highlighted: result.text.replace(
        regex,
        '<mark>$1</mark>'
      )
    }));
  }, [query, results]);

  return (
    <ul>
      {highlightedResults.map(result => (
        <li
          key={result.id}
          dangerouslySetInnerHTML={{ __html: result.highlighted }}
        />
      ))}
    </ul>
  );
}
```

---

### Form Validation (useRef + useMemo)

```typescript
function Form() {
  const formRef = useRef<HTMLFormElement>(null);
  const [values, setValues] = useState({ email: '', password: '' });

  const errors = useMemo(() => {
    const errors: Record<string, string> = {};

    if (values.email && !values.email.includes('@')) {
      errors.email = 'Invalid email';
    }

    if (values.password && values.password.length < 8) {
      errors.password = 'Password must be 8+ characters';
    }

    return errors;
  }, [values]);

  const isValid = Object.keys(errors).length === 0;

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    if (isValid) {
      console.log('Submitted:', values);
      formRef.current?.reset();
    }
  };

  return (
    <form ref={formRef} onSubmit={handleSubmit}>
      <input
        type="email"
        value={values.email}
        onChange={e => setValues(v => ({ ...v, email: e.target.value }))}
      />
      {errors.email && <span>{errors.email}</span>}

      <input
        type="password"
        value={values.password}
        onChange={e => setValues(v => ({ ...v, password: e.target.value }))}
      />
      {errors.password && <span>{errors.password}</span>}

      <button disabled={!isValid}>Submit</button>
    </form>
  );
}
```

---

## ❓ 7. Interview Questions

### Q1: useRef vs useState - when to use each?

**Answer:**

| Feature                     | useRef                    | useState               |
| --------------------------- | ------------------------- | ---------------------- |
| **Triggers re-render**      | ❌ No                     | ✅ Yes                 |
| **Persists across renders** | ✅ Yes                    | ✅ Yes                 |
| **Update timing**           | Immediate                 | Next render            |
| **Use cases**               | DOM access, timers, flags | UI state, data display |

```typescript
// ❌ BAD: Timer ID in state (unnecessary re-renders)
const [timerId, setTimerId] = useState<number | null>(null);

// ✅ GOOD: Timer ID in ref (no re-renders)
const timerIdRef = useRef<number | null>(null);

// ❌ BAD: User input in ref (UI doesn't update)
const inputRef = useRef("");

// ✅ GOOD: User input in state (UI updates)
const [input, setInput] = useState("");
```

---

### Q2: Why use useMemo for objects/arrays passed to child components?

**Answer:**

**Without useMemo:**

```typescript
function Parent() {
  const [count, setCount] = useState(0);

  const config = { theme: 'dark' }; // New object every render!

  return <Child config={config} />; // Child re-renders!
}

const Child = memo(({ config }) => {
  // Re-renders even though config values unchanged
  // because config is a new object reference
});
```

**With useMemo:**

```typescript
function Parent() {
  const [count, setCount] = useState(0);

  const config = useMemo(
    () => ({ theme: 'dark' }),
    [] // Same object reference!
  );

  return <Child config={config} />; // Child doesn't re-render!
}
```

**React.memo compares props with `Object.is()`:**

- Primitives: `5 === 5` ✅
- Objects: `{} === {}` ❌ (different references)
- Memoized objects: Same reference ✅

---

### Q3: Can you show useMemo vs useCallback difference?

**Answer:**

```typescript
// useMemo: Memoize VALUE
const expensiveValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);

// useCallback: Memoize FUNCTION
const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);

// Equivalent!
const memoizedCallback = useMemo(
  () => () => {
    doSomething(a, b);
  },
  [a, b],
);
```

**Use useCallback when:**

- Passing callback to memoized child
- Callback is dependency of useEffect

**Use useMemo when:**

- Caching expensive computation result
- Maintaining object/array reference

---

## 🎯 Summary

**useRef:**

- Mutable container (`{ current: value }`)
- No re-renders when mutated
- Use for: DOM refs, timers, previous values, instance variables

**useMemo:**

- Cache expensive computations
- Recompute only when dependencies change
- Use for: Large array operations, derived state, referential equality

**Best practices:**

- ✅ useRef for values that shouldn't trigger re-renders
- ✅ useMemo for expensive operations (>10ms)
- ✅ useMemo for referential equality with `React.memo`
- ❌ Don't premature-optimize with useMemo

Master these hooks for optimal React performance! 🎯
