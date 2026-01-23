# ⚛️ React Interview Questions (Senior Level)

> Senior React interview questions covering hooks, performance, internals, and architectural decisions. Based on actual FAANG interviews.

---

## 🎯 Part 1: React Fundamentals

### Q1: Explain React's reconciliation algorithm

**Answer:**

**Reconciliation** is how React updates the DOM efficiently by comparing virtual DOM trees.

**Diffing algorithm assumptions:**

1. **Different types → replace entire tree**
2. **Same type → update props**
3. **Keys → identify stable elements**

**Example 1: Type change**

```jsx
// Before
<div>
  <Counter />
</div>

// After
<span>
  <Counter />
</span>

// Result: Counter unmounts & remounts (new instance!)
// State is LOST
```

**Example 2: Props change**

```jsx
// Before
<Button color="blue" />

// After
<Button color="red" />

// Result: Same instance, props updated
// State PRESERVED
```

**Example 3: Keys for list reconciliation**

```jsx
// ❌ WITHOUT keys (index as key)
{
  items.map((item, index) => <Item key={index} {...item} />);
}

// Problem: Reordering causes inefficient updates
// [A, B, C] → [C, A, B]
// React thinks:
// - A changed to C (update)
// - B changed to A (update)
// - C changed to B (update)
// = 3 updates!

// ✅ WITH stable keys
{
  items.map((item) => <Item key={item.id} {...item} />);
}

// [A, B, C] → [C, A, B]
// React knows:
// - Move C to front
// = 1 move operation!
```

**Why keys matter in production:**

```jsx
// ❌ BAD: Causes input to lose focus on reorder
{
  todos.map((todo, index) => <TodoItem key={index} todo={todo} />);
}

// User types in second todo
// List reorders
// Focus jumps to wrong input!

// ✅ GOOD: Stable key preserves component identity
{
  todos.map((todo) => <TodoItem key={todo.id} todo={todo} />);
}
```

---

### Q2: What's the difference between controlled and uncontrolled components?

**Answer:**

| Controlled                     | Uncontrolled           |
| ------------------------------ | ---------------------- |
| React state as source of truth | DOM as source of truth |
| Value prop + onChange          | ref to access value    |
| Predictable                    | Less predictable       |
| Can validate on each keystroke | Validation on submit   |

**Controlled component:**

```jsx
function ControlledInput() {
  const [value, setValue] = useState("");

  return (
    <input
      value={value} // React controls value
      onChange={(e) => setValue(e.target.value)}
    />
  );
}

// ✅ Benefits:
// - Instant validation
// - Conditional rendering
// - Format input (e.g., phone number)
```

**Uncontrolled component:**

```jsx
function UncontrolledInput() {
  const inputRef = useRef(null);

  function handleSubmit() {
    console.log(inputRef.current.value); // Read from DOM
  }

  return <input ref={inputRef} defaultValue="initial" />;
}

// ✅ Benefits:
// - Less re-renders
// - Simpler for large forms
// - Third-party libraries integration
```

**Real-world example: Form with validation**

```jsx
function SignupForm() {
  const [email, setEmail] = useState("");
  const [emailError, setEmailError] = useState("");

  function validateEmail(value) {
    if (!value.includes("@")) {
      setEmailError("Invalid email");
    } else {
      setEmailError("");
    }
  }

  return (
    <div>
      <input
        type="email"
        value={email}
        onChange={(e) => {
          setEmail(e.target.value);
          validateEmail(e.target.value); // Instant validation
        }}
      />
      {emailError && <span className="error">{emailError}</span>}
    </div>
  );
}
```

**When to use which:**

- **Controlled**: Forms with validation, formatting, dynamic fields
- **Uncontrolled**: Simple forms, file inputs, third-party integrations

---

### Q3: How does React Fiber work?

**Answer:**

**Fiber** is React's reconciliation engine that enables:

- **Incremental rendering** (break work into chunks)
- **Pause/resume** work
- **Prioritize updates** (urgent vs non-urgent)

**Before Fiber (React 15):**

```
Start render → [Do ALL work] → Commit to DOM
                 ↑
            Blocks main thread!
            (Can't cancel or pause)
```

**With Fiber (React 16+):**

```
Start render → [Work chunk 1] → Yield to browser
            → [Work chunk 2] → Yield
            → [Work chunk 3] → Commit to DOM
               ↑
           Can pause/cancel!
```

**Fiber architecture:**

```javascript
// Fiber node structure (simplified)
{
  type: 'div',           // Component type
  props: { ... },        // Props
  stateNode: domNode,    // Actual DOM node
  child: fiberNode,      // First child
  sibling: fiberNode,    // Next sibling
  return: fiberNode,     // Parent
  alternate: fiberNode,  // Previous version
  effectTag: 'UPDATE',   // What changed
  memoizedState: { ... } // Hooks state
}
```

**Priority lanes (React 18):**

```javascript
// Urgent: User input (clicks, typing)
startTransition(() => {
  // Non-urgent: Heavy computation
  setSearchResults(expensiveFilter(data));
});

// React prioritizes user input over search results
```

**Real-world impact:**

```jsx
function App() {
  const [text, setText] = useState("");
  const [items, setItems] = useState(generateItems(10000));

  // ❌ Without Fiber: Input lags (blocks main thread)

  // ✅ With Fiber: Input responsive
  // React pauses item rendering to handle input
}
```

**Follow-up: What's time slicing?**

React breaks work into 5ms chunks, yielding to browser between chunks for:

- User input
- Animations
- Other high-priority tasks

---

## 🎯 Part 2: Hooks

### Q4: How does useState work internally?

**Answer:**

**useState uses a hook linked list stored in Fiber node.**

**Simplified implementation:**

```javascript
let currentFiber = null;
let hookIndex = 0;

function useState(initialValue) {
  const fiber = currentFiber;
  const hooks = fiber.memoizedState || [];

  if (hooks[hookIndex] === undefined) {
    // First render: Initialize
    hooks[hookIndex] = initialValue;
  }

  const currentIndex = hookIndex;
  const setState = (newValue) => {
    hooks[currentIndex] = newValue;
    scheduleRerender();
  };

  const value = hooks[hookIndex];
  hookIndex++;

  return [value, setState];
}
```

**Why hook order matters:**

```jsx
// ❌ WRONG: Conditional hook
function Component({ condition }) {
  if (condition) {
    const [state, setState] = useState(0); // Hook #1 sometimes
  }
  const [count, setCount] = useState(0); // Hook #1 or #2?

  // Hook order changes → breaks hook linked list!
}

// ✅ CORRECT: Same order every render
function Component({ condition }) {
  const [state, setState] = useState(0); // Always hook #1
  const [count, setCount] = useState(0); // Always hook #2

  if (condition) {
    // Use state conditionally, don't declare conditionally
  }
}
```

**State batching:**

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1); // count = 0
    setCount(count + 1); // count = 0
    setCount(count + 1); // count = 0
    // Result: count = 1 (not 3!)

    // Batched into single update
  }

  // ✅ Use functional updates
  function handleClickCorrect() {
    setCount((c) => c + 1); // 0 + 1 = 1
    setCount((c) => c + 1); // 1 + 1 = 2
    setCount((c) => c + 1); // 2 + 1 = 3
    // Result: count = 3
  }
}
```

**React 18 automatic batching:**

```jsx
// React 17: Not batched
setTimeout(() => {
  setCount((c) => c + 1); // Render
  setFlag((f) => !f); // Render again
}, 1000);

// React 18: Batched everywhere
setTimeout(() => {
  setCount((c) => c + 1); // Batched
  setFlag((f) => !f); // Batched
  // Single render
}, 1000);
```

---

### Q5: When should you use useCallback and useMemo?

**Answer:**

**Only optimize when you have a measured performance problem!**

**useCallback: Memoize function reference**

```jsx
// ❌ PREMATURE: Unnecessary useCallback
function Parent() {
  const handleClick = useCallback(() => {
    console.log("Clicked");
  }, []); // Overhead for no benefit

  return <div onClick={handleClick}>Click</div>;
}

// ✅ NECESSARY: Child is memoized
const Child = memo(({ onClick }) => {
  console.log("Child render");
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const [count, setCount] = useState(0);

  // Without useCallback: Child re-renders on every Parent render
  // With useCallback: Child only renders when onClick changes
  const handleClick = useCallback(() => {
    console.log("Clicked");
  }, []);

  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>Count: {count}</button>
      <Child onClick={handleClick} />
    </div>
  );
}
```

**useMemo: Memoize expensive calculation**

```jsx
// ❌ PREMATURE: Cheap calculation
function Component({ items }) {
  const count = useMemo(() => items.length, [items]);
  // Overhead > benefit (length is O(1))
}

// ✅ NECESSARY: Expensive calculation
function Component({ items }) {
  const sortedItems = useMemo(() => {
    return [...items].sort((a, b) => {
      // Complex sorting logic
      return expensiveComparison(a, b);
    });
  }, [items]);

  return <List items={sortedItems} />;
}
```

**Real-world example: Filtered list**

```jsx
function SearchableList({ items, searchQuery }) {
  // ❌ Filters on every render (even when count changes)
  const filteredItems = items.filter((item) =>
    item.name.toLowerCase().includes(searchQuery.toLowerCase()),
  );

  // ✅ Only recomputes when items or searchQuery changes
  const filteredItems = useMemo(() => {
    return items.filter((item) =>
      item.name.toLowerCase().includes(searchQuery.toLowerCase()),
    );
  }, [items, searchQuery]);

  return <List items={filteredItems} />;
}
```

**When to use:**

- **useCallback**: Passing to memoized child or useEffect dependency
- **useMemo**: Expensive calculations or referential equality for deps

**When NOT to use:**

- Premature optimization
- Cheap calculations
- Non-memoized children

---

### Q6: Explain useEffect cleanup and timing

**Answer:**

**useEffect runs AFTER render & DOM update.**

**Lifecycle timing:**

```jsx
function Component() {
  useEffect(() => {
    console.log("Effect runs");

    return () => {
      console.log("Cleanup runs");
    };
  });

  console.log("Render");
}

// First mount:
// 1. 'Render'
// 2. DOM updates
// 3. 'Effect runs'

// Re-render:
// 1. 'Render'
// 2. DOM updates
// 3. 'Cleanup runs' (previous effect)
// 4. 'Effect runs' (new effect)

// Unmount:
// 1. 'Cleanup runs'
```

**Cleanup use cases:**

**1. Event listeners:**

```jsx
useEffect(() => {
  function handleResize() {
    setWidth(window.innerWidth);
  }

  window.addEventListener("resize", handleResize);

  // ✅ Cleanup: Remove listener
  return () => {
    window.removeEventListener("resize", handleResize);
  };
}, []);
```

**2. Timers:**

```jsx
useEffect(() => {
  const timer = setInterval(() => {
    setCount((c) => c + 1);
  }, 1000);

  // ✅ Cleanup: Clear timer
  return () => {
    clearInterval(timer);
  };
}, []);
```

**3. Subscriptions:**

```jsx
useEffect(() => {
  const subscription = api.subscribe((data) => {
    setData(data);
  });

  // ✅ Cleanup: Unsubscribe
  return () => {
    subscription.unsubscribe();
  };
}, []);
```

**4. Abort API calls:**

```jsx
useEffect(() => {
  const controller = new AbortController();

  async function fetchData() {
    try {
      const res = await fetch("/api/data", {
        signal: controller.signal,
      });
      const data = await res.json();
      setData(data);
    } catch (error) {
      if (error.name !== "AbortError") {
        console.error(error);
      }
    }
  }

  fetchData();

  // ✅ Cleanup: Cancel request
  return () => {
    controller.abort();
  };
}, []);
```

**Common mistake: Missing cleanup**

```jsx
// ❌ Memory leak
function Component() {
  useEffect(() => {
    setInterval(() => {
      console.log("Running...");
    }, 1000);
    // No cleanup! Timer keeps running after unmount
  }, []);
}

// ✅ Cleaned up
function Component() {
  useEffect(() => {
    const timer = setInterval(() => {
      console.log("Running...");
    }, 1000);

    return () => clearInterval(timer);
  }, []);
}
```

---

## 🎯 Part 3: Performance

### Q7: How do you optimize React performance?

**Answer:**

**1. Profiling first (measure before optimizing):**

```javascript
// React DevTools Profiler
// Identify slow components

// Or programmatic profiling
import { Profiler } from "react";

<Profiler
  id="MyComponent"
  onRender={(id, phase, actualDuration) => {
    console.log(`${id} (${phase}) took ${actualDuration}ms`);
  }}
>
  <MyComponent />
</Profiler>;
```

**2. React.memo (prevent unnecessary re-renders):**

```jsx
// ❌ Child re-renders on every parent render
function Child({ value }) {
  console.log("Child render");
  return <div>{value}</div>;
}

// ✅ Child only re-renders when props change
const Child = memo(function Child({ value }) {
  console.log("Child render");
  return <div>{value}</div>;
});

// Custom comparison
const Child = memo(Child, (prevProps, nextProps) => {
  // Return true if props equal (skip render)
  return prevProps.value === nextProps.value;
});
```

**3. Code splitting:**

```jsx
// ❌ Load everything upfront
import HeavyComponent from "./HeavyComponent";

// ✅ Lazy load
const HeavyComponent = lazy(() => import("./HeavyComponent"));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <HeavyComponent />
    </Suspense>
  );
}
```

**4. Virtualization (render only visible items):**

```jsx
import { FixedSizeList } from "react-window";

function VirtualizedList({ items }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {({ index, style }) => <div style={style}>{items[index].name}</div>}
    </FixedSizeList>
  );
}

// Renders only visible items (~20) instead of all (10,000+)
```

**5. Debounce expensive operations:**

```jsx
import { debounce } from "lodash";

function SearchBox() {
  const [query, setQuery] = useState("");

  const debouncedSearch = useMemo(
    () =>
      debounce((q) => {
        // Expensive search
        fetchResults(q);
      }, 300),
    [],
  );

  useEffect(() => {
    return () => debouncedSearch.cancel();
  }, [debouncedSearch]);

  return (
    <input
      onChange={(e) => {
        setQuery(e.target.value);
        debouncedSearch(e.target.value);
      }}
    />
  );
}
```

**6. useMemo for expensive calculations:**

```jsx
function Component({ items, filter }) {
  const filteredItems = useMemo(() => {
    return items.filter((item) => {
      return expensiveCheck(item, filter);
    });
  }, [items, filter]);

  return <List items={filteredItems} />;
}
```

---

### Q8: What causes unnecessary re-renders?

**Answer:**

**1. Parent re-renders → All children re-render**

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>Count: {count}</button>
      <ExpensiveChild /> {/* Re-renders even though props didn't change! */}
    </div>
  );
}

// ✅ Fix: Memoize child
const ExpensiveChild = memo(function ExpensiveChild() {
  // Only re-renders when props change
});
```

**2. Inline object/array creation**

```jsx
function Parent() {
  return (
    <Child
      style={{ color: "red" }} // New object every render!
      items={[1, 2, 3]} // New array every render!
    />
  );
}

// ✅ Fix: Move outside or useMemo
const style = { color: "red" };
const items = [1, 2, 3];

function Parent() {
  return <Child style={style} items={items} />;
}
```

**3. Inline function creation**

```jsx
function Parent() {
  return (
    <Child
      onClick={() => console.log("Click")} // New function every render!
    />
  );
}

// ✅ Fix: useCallback
function Parent() {
  const handleClick = useCallback(() => {
    console.log("Click");
  }, []);

  return <Child onClick={handleClick} />;
}
```

**4. Context changes**

```jsx
const Context = createContext();

function Provider({ children }) {
  const [user, setUser] = useState(null);

  // ❌ New object every render → All consumers re-render!
  return (
    <Context.Provider value={{ user, setUser }}>{children}</Context.Provider>
  );
}

// ✅ Fix: Memoize value
function Provider({ children }) {
  const [user, setUser] = useState(null);

  const value = useMemo(() => ({ user, setUser }), [user]);

  return <Context.Provider value={value}>{children}</Context.Provider>;
}
```

---

## 🎯 Summary

**React core concepts:**

- ✅ Reconciliation (diffing, keys)
- ✅ Controlled vs uncontrolled
- ✅ Fiber architecture
- ✅ Hook internals & rules
- ✅ useEffect cleanup

**Performance:**

- ✅ React.memo for pure components
- ✅ useCallback for stable functions
- ✅ useMemo for expensive calculations
- ✅ Code splitting with lazy()
- ✅ Virtualization for long lists

**Common pitfalls:**

- ❌ Index as key
- ❌ Inline objects/functions
- ❌ Conditional hooks
- ❌ Missing useEffect cleanup
- ❌ Premature optimization

Practice explaining with code examples! 💪
