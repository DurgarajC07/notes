# ⚡ useCallback Performance Deep Dive

> `useCallback` memoizes function references to prevent unnecessary re-renders of child components. Master this hook to optimize React performance and avoid common pitfalls.

---

## 📖 1. Concept Explanation

### What is useCallback?

**Memoize function references, return same function instance unless dependencies change:**

```typescript
const memoizedCallback = useCallback(
  () => {
    doSomething(a, b);
  },
  [a, b], // Dependencies
);
```

**Equivalent to:**

```typescript
const memoizedCallback = useMemo(
  () => () => {
    doSomething(a, b);
  },
  [a, b],
);
```

---

## 🧠 2. Why It Matters

### Problem: Functions Create New References

```typescript
// ❌ BAD: New function every render
function Parent() {
  const [count, setCount] = useState(0);

  // New function reference on every render!
  const handleClick = () => {
    console.log('Clicked');
  };

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>
        Count: {count}
      </button>
      <ExpensiveChild onClick={handleClick} /> {/* Re-renders! */}
    </>
  );
}

const ExpensiveChild = React.memo(({ onClick }) => {
  console.log('ExpensiveChild rendered');
  return <button onClick={onClick}>Child Button</button>;
});
```

**Every time Parent renders:**

1. New `handleClick` function created
2. `ExpensiveChild` receives new `onClick` prop
3. `React.memo` sees different reference → re-renders
4. Even though logic is identical!

---

### Solution: useCallback

```typescript
// ✅ GOOD: Same function reference
function Parent() {
  const [count, setCount] = useState(0);

  // Same function reference across renders!
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []); // No dependencies

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>
        Count: {count}
      </button>
      <ExpensiveChild onClick={handleClick} /> {/* No re-render! */}
    </>
  );
}
```

**Result:**

- `handleClick` reference stays the same
- `ExpensiveChild` doesn't re-render when `count` changes
- Performance improvement! ⚡

---

## ⚙️ 3. When to Use useCallback

### 1. Passing Callbacks to Memoized Components

```typescript
import { memo, useCallback } from 'react';

interface ChildProps {
  onSave: (data: string) => void;
}

// Memoized child component
const ExpensiveForm = memo<ChildProps>(({ onSave }) => {
  console.log('ExpensiveForm rendered');

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      onSave('data');
    }}>
      {/* Complex form fields... */}
      <button type="submit">Save</button>
    </form>
  );
});

// ❌ BAD: New function every render
function Parent() {
  const [items, setItems] = useState<string[]>([]);

  const handleSave = (data: string) => {
    setItems(prev => [...prev, data]);
  };

  return <ExpensiveForm onSave={handleSave} />; // Re-renders every time!
}

// ✅ GOOD: Memoized callback
function Parent() {
  const [items, setItems] = useState<string[]>([]);

  const handleSave = useCallback((data: string) => {
    setItems(prev => [...prev, data]);
  }, []); // Stable reference

  return <ExpensiveForm onSave={handleSave} />; // No unnecessary re-renders!
}
```

---

### 2. useEffect Dependencies

```typescript
// ❌ BAD: Infinite loop
function SearchComponent({ apiUrl }: { apiUrl: string }) {
  const [results, setResults] = useState([]);

  const fetchData = () => {
    fetch(apiUrl)
      .then((res) => res.json())
      .then(setResults);
  };

  useEffect(() => {
    fetchData();
  }, [fetchData]); // fetchData changes every render → infinite loop!
}

// ✅ GOOD: Stable function reference
function SearchComponent({ apiUrl }: { apiUrl: string }) {
  const [results, setResults] = useState([]);

  const fetchData = useCallback(() => {
    fetch(apiUrl)
      .then((res) => res.json())
      .then(setResults);
  }, [apiUrl]); // Only changes when apiUrl changes

  useEffect(() => {
    fetchData();
  }, [fetchData]); // Safe dependency
}
```

---

### 3. Event Handlers with Dependencies

```typescript
function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all');

  // ❌ BAD: New function every render (even when filter unchanged)
  const handleToggle = (id: number) => {
    setTodos(todos =>
      todos.map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
  };

  // ✅ GOOD: Memoized (no dependencies!)
  const handleToggleMemoized = useCallback((id: number) => {
    setTodos(todos =>
      todos.map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
  }, []); // Functional update → no dependency on todos

  const filteredTodos = useMemo(() => {
    switch (filter) {
      case 'active': return todos.filter(t => !t.completed);
      case 'completed': return todos.filter(t => t.completed);
      default: return todos;
    }
  }, [todos, filter]);

  return (
    <>
      <FilterButtons filter={filter} setFilter={setFilter} />
      <TodoItems items={filteredTodos} onToggle={handleToggleMemoized} />
    </>
  );
}
```

---

## ❌ 4. When NOT to Use useCallback

### 1. Simple Event Handlers (No Memo)

```typescript
// ❌ Over-optimization
function Counter() {
  const [count, setCount] = useState(0);

  const handleIncrement = useCallback(() => {
    setCount(c => c + 1);
  }, []); // Unnecessary!

  return <button onClick={handleIncrement}>{count}</button>;
}

// ✅ Just use inline function
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      {count}
    </button>
  );
}
```

**Why?**

- No child component to optimize
- useCallback overhead > new function cost

---

### 2. Non-Memoized Children

```typescript
// ❌ Pointless: Child not memoized
function Parent() {
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []);

  return <Child onClick={handleClick} />; // Child re-renders anyway!
}

// Child not wrapped in React.memo
function Child({ onClick }) {
  return <button onClick={onClick}>Click</button>;
}

// ✅ Either memoize child or skip useCallback
const MemoizedChild = React.memo(Child);

function Parent() {
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []);

  return <MemoizedChild onClick={handleClick} />;
}
```

---

### 3. Dependencies Change Frequently

```typescript
// ❌ Pointless: Dependencies change every render anyway
function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  const handleSearch = useCallback(() => {
    fetch(`/api/search?q=${query}`)
      .then((res) => res.json())
      .then(setResults);
  }, [query]); // query changes every keystroke!

  // No benefit: function recreated anyway
}

// ✅ Better: Debounce the query instead
function Search() {
  const [query, setQuery] = useState("");
  const [debouncedQuery, setDebouncedQuery] = useState("");

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedQuery(query);
    }, 300);

    return () => clearTimeout(timer);
  }, [query]);

  const handleSearch = useCallback(() => {
    fetch(`/api/search?q=${debouncedQuery}`)
      .then((res) => res.json())
      .then(setResults);
  }, [debouncedQuery]); // Only changes after debounce
}
```

---

## ✅ 5. Common Pitfalls

### Pitfall 1: Stale Closures

```typescript
// ❌ BAD: Stale closure
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {
    console.log(count); // Always logs 0!
  }, []); // Empty deps → closure captures initial count

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <button onClick={handleClick}>Log Count</button>
    </>
  );
}

// ✅ FIX 1: Include dependency
const handleClick = useCallback(() => {
  console.log(count);
}, [count]); // Recreates when count changes

// ✅ FIX 2: Use ref for latest value
const countRef = useRef(count);
countRef.current = count;

const handleClick = useCallback(() => {
  console.log(countRef.current); // Always latest!
}, []); // Stable, but reads latest via ref
```

---

### Pitfall 2: Object/Array Dependencies

```typescript
// ❌ BAD: Object dependency changes every render
function Parent({ config }) {
  const handleSave = useCallback(
    (data) => {
      fetch(config.apiUrl, {
        // config is new object every render!
        method: "POST",
        body: JSON.stringify(data),
      });
    },
    [config],
  ); // Function recreated every render anyway
}

// ✅ FIX: Depend on specific properties
function Parent({ config }) {
  const { apiUrl } = config;

  const handleSave = useCallback(
    (data) => {
      fetch(apiUrl, {
        method: "POST",
        body: JSON.stringify(data),
      });
    },
    [apiUrl],
  ); // Only recreates when apiUrl changes
}
```

---

### Pitfall 3: Missing Dependencies

```typescript
// ❌ BAD: Missing dependency
function TodoList({ userId }) {
  const [todos, setTodos] = useState([]);

  const fetchTodos = useCallback(() => {
    fetch(`/api/users/${userId}/todos`) // Uses userId!
      .then((res) => res.json())
      .then(setTodos);
  }, []); // Missing userId dependency!

  // userId changes but fetchTodos doesn't update
}

// ✅ FIX: Include all dependencies
const fetchTodos = useCallback(() => {
  fetch(`/api/users/${userId}/todos`)
    .then((res) => res.json())
    .then(setTodos);
}, [userId]); // Recreates when userId changes
```

---

## 🧪 6. Real-World Examples

### Debounced Search

```typescript
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const debouncedQuery = useDebounce(query, 300);

  const fetchResults = useCallback(async (searchQuery: string) => {
    if (!searchQuery) {
      setResults([]);
      return;
    }

    const res = await fetch(`/api/search?q=${searchQuery}`);
    const data = await res.json();
    setResults(data);
  }, []);

  useEffect(() => {
    fetchResults(debouncedQuery);
  }, [debouncedQuery, fetchResults]);

  return (
    <>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <SearchResults results={results} />
    </>
  );
}
```

---

### Virtualized List with Callbacks

```typescript
interface Item {
  id: number;
  name: string;
}

const VirtualizedRow = memo<{
  item: Item;
  onEdit: (id: number) => void;
  onDelete: (id: number) => void;
}>(({ item, onEdit, onDelete }) => {
  console.log(`Rendering row ${item.id}`);

  return (
    <div>
      <span>{item.name}</span>
      <button onClick={() => onEdit(item.id)}>Edit</button>
      <button onClick={() => onDelete(item.id)}>Delete</button>
    </div>
  );
});

function VirtualList({ items }: { items: Item[] }) {
  const [editingId, setEditingId] = useState<number | null>(null);

  // ✅ Memoized callbacks prevent all rows from re-rendering
  const handleEdit = useCallback((id: number) => {
    setEditingId(id);
  }, []);

  const handleDelete = useCallback((id: number) => {
    // API call to delete
    console.log('Deleting', id);
  }, []);

  return (
    <div>
      {items.map(item => (
        <VirtualizedRow
          key={item.id}
          item={item}
          onEdit={handleEdit}
          onDelete={handleDelete}
        />
      ))}
    </div>
  );
}
```

---

### Form with Validation

```typescript
interface FormValues {
  email: string;
  password: string;
}

interface FormErrors {
  email?: string;
  password?: string;
}

function LoginForm() {
  const [values, setValues] = useState<FormValues>({
    email: '',
    password: ''
  });
  const [errors, setErrors] = useState<FormErrors>({});

  const validate = useCallback((values: FormValues): FormErrors => {
    const errors: FormErrors = {};

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
  }, []);

  const handleSubmit = useCallback((e: React.FormEvent) => {
    e.preventDefault();

    const validationErrors = validate(values);
    setErrors(validationErrors);

    if (Object.keys(validationErrors).length === 0) {
      console.log('Submitting:', values);
    }
  }, [values, validate]);

  const handleChange = useCallback((field: keyof FormValues) => {
    return (e: React.ChangeEvent<HTMLInputElement>) => {
      setValues(prev => ({ ...prev, [field]: e.target.value }));
    };
  }, []);

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={values.email}
        onChange={handleChange('email')}
      />
      {errors.email && <span>{errors.email}</span>}

      <input
        type="password"
        value={values.password}
        onChange={handleChange('password')}
      />
      {errors.password && <span>{errors.password}</span>}

      <button type="submit">Login</button>
    </form>
  );
}
```

---

## ❓ 7. Interview Questions

### Q1: useCallback vs useMemo?

**Answer:**

```typescript
// useCallback: Memoize FUNCTION
const memoizedFunc = useCallback(() => doSomething(a, b), [a, b]);

// useMemo: Memoize RETURN VALUE
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);

// Equivalent:
const memoizedFunc = useMemo(() => () => doSomething(a, b), [a, b]);
```

**When to use:**

- **useCallback:** Passing callbacks to memoized children, useEffect deps
- **useMemo:** Expensive computations, object/array referential equality

---

### Q2: Why doesn't useCallback always prevent re-renders?

**Answer:**

**useCallback only helps if:**

1. ✅ Child is memoized with `React.memo()`
2. ✅ Callback is in child's props
3. ✅ Dependencies don't change every render

```typescript
// ❌ Doesn't help: Child not memoized
function Parent() {
  const handleClick = useCallback(() => {}, []);
  return <Child onClick={handleClick} />; // Re-renders anyway
}

function Child({ onClick }) {
  return <button onClick={onClick}>Click</button>;
}

// ✅ Helps: Child memoized
const MemoChild = memo(({ onClick }) => {
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const handleClick = useCallback(() => {}, []);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <MemoChild onClick={handleClick} /> {/* Doesn't re-render! */}
    </>
  );
}
```

---

### Q3: How to avoid stale closures with useCallback?

**Answer:**

**3 solutions:**

**1. Include dependencies:**

```typescript
const handleClick = useCallback(() => {
  console.log(count); // Fresh value
}, [count]); // Recreates when count changes
```

**2. Use ref for latest value:**

```typescript
const countRef = useRef(count);
countRef.current = count;

const handleClick = useCallback(() => {
  console.log(countRef.current); // Always latest
}, []); // Stable reference
```

**3. Functional updates:**

```typescript
const handleIncrement = useCallback(() => {
  setCount((c) => c + 1); // Uses latest value
}, []); // No dependency on count
```

---

## 🎯 Summary

**useCallback:**

- Memoize function references
- Prevent child re-renders (with React.memo)
- Stabilize useEffect dependencies

**When to use:**

- ✅ Passing callbacks to memoized children
- ✅ useEffect dependencies
- ✅ Event handlers with dependencies

**When NOT to use:**

- ❌ Simple event handlers (no memo)
- ❌ Non-memoized children
- ❌ Dependencies change frequently

**Common pitfalls:**

- Stale closures (missing dependencies)
- Object/array dependencies
- Over-optimization

Master useCallback for optimal React performance! ⚡
