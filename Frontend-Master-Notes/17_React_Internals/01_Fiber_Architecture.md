# ⚛️ React Fiber Architecture

> React Fiber is the reimplementation of React's core algorithm that enables incremental rendering, prioritization, and time-slicing. Understanding Fiber is crucial for senior engineers building high-performance React applications.

---

## 📖 1. Concept Explanation

### What is React Fiber?

**Fiber** is React's reconciliation engine (introduced in React 16). It's a complete rewrite of the previous stack-based reconciler that enables:

1. **Incremental Rendering** - Split rendering work into chunks
2. **Prioritization** - Assign priority to different types of updates
3. **Pause and Resume** - Pause work and resume later
4. **Abort Work** - Discard work if no longer needed
5. **Reuse Completed Work** - Optimize re-renders

### Why Fiber Was Created

**Problem with Old Stack Reconciler (React 15):**

```javascript
// Synchronous, blocking rendering
function renderTree(component) {
  // Process entire tree in one go
  // Can't pause or split work
  // Browser blocked until complete
}
```

**Issues:**

- Large component trees caused frame drops (animations janky)
- No way to prioritize urgent updates (user input) over less urgent (data fetch)
- All rendering synchronous and blocking
- 60fps target often missed (16.67ms per frame budget exceeded)

**Fiber's Solution:**

```javascript
// Asynchronous, interruptible rendering
function performUnitOfWork(fiber) {
  // Process one fiber node
  // Check if time remaining
  // If not, yield to browser
  // Resume later
}
```

---

## 🧠 2. Why It Matters in Real Projects

### Production Impact

**1. Smooth UIs:**

```javascript
// Without Fiber: Typing feels laggy
function HeavyList({ items }) {
  return (
    <div>
      {items.map((item) => (
        <HeavyComponent key={item.id} data={item} />
      ))}
      {/* Renders all 10,000 items synchronously */}
    </div>
  );
}

// With Fiber: Typing remains smooth
// React can pause rendering heavy list to handle input
```

**2. Better UX for Complex Apps:**

- Chat apps: Message input responsive while rendering message history
- Dashboards: Charts load without blocking filters
- E-commerce: Search autocomplete doesn't freeze during product grid render

**3. Concurrent Features (React 18+):**

- `useTransition` - Mark updates as non-urgent
- `useDeferredValue` - Defer expensive computations
- Suspense for data fetching
- Automatic batching

**4. Developer Experience:**

- Better error boundaries
- Improved stack traces
- DevTools time-travel debugging

---

## ⚙️ 3. Internal Working

### Fiber Data Structure

Each React element has a corresponding **Fiber node**:

```javascript
// Simplified Fiber structure
{
  type: 'div',              // Component type
  key: null,                // Key for reconciliation
  props: { className: 'container' },

  // Parent-child relationships
  return: parentFiber,      // Parent fiber
  child: firstChildFiber,   // First child
  sibling: nextSiblingFiber, // Next sibling

  // State
  memoizedState: null,      // Current state
  memoizedProps: null,      // Current props

  // Effects
  flags: Update | Placement, // What needs to be done

  // Scheduling
  lanes: InputContinuousLane, // Priority

  // Work in progress
  alternate: currentFiber,   // Reference to current tree

  // Other
  stateNode: domNode,       // Actual DOM reference
  updateQueue: null         // Pending updates
}
```

### Double Buffering (current vs workInProgress)

React maintains **two fiber trees**:

1. **current** - What's displayed on screen
2. **workInProgress** - Work being computed

```javascript
// Render phase
workInProgress = createWorkInProgress(current);

// Commit phase
current = workInProgress;
workInProgress = null;

// Next render
workInProgress = createWorkInProgress(current);
// Reuses fibers from previous render
```

**Benefits:**

- Can abandon incomplete work without affecting displayed UI
- Reuse completed work
- Compare old and new trees efficiently

### Fiber Tree Structure

```javascript
// React element tree:
<App>
  <Header />
  <Main>
    <Sidebar />
    <Content />
  </Main>
</App>

// Fiber tree (linked list structure):
App
 └─ child → Header
            └─ sibling → Main
                         └─ child → Sidebar
                                    └─ sibling → Content

// Traversal:
function performWorkOnFiber(fiber) {
  // 1. Process current fiber
  beginWork(fiber);

  // 2. If has child, go to child
  if (fiber.child) {
    return fiber.child;
  }

  // 3. If no child, go to sibling
  while (fiber) {
    completeWork(fiber);
    if (fiber.sibling) {
      return fiber.sibling;
    }
    // 4. If no sibling, go to parent
    fiber = fiber.return;
  }
}
```

### Render and Commit Phases

**Render Phase** (Interruptible):

```javascript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

// Can be paused and resumed
```

**Commit Phase** (Synchronous):

```javascript
function commitRoot(root) {
  // 1. Before mutation (getSnapshotBeforeUpdate)
  commitBeforeMutationEffects(root);

  // 2. Mutation (DOM updates)
  commitMutationEffects(root);

  // 3. Layout effects (useLayoutEffect, componentDidMount)
  commitLayoutEffects(root);

  // 4. Passive effects (useEffect)
  schedulePassiveEffects(root);
}

// Cannot be interrupted - must complete atomically
```

### Priority Levels (Lanes)

React assigns different priorities to updates:

```javascript
// Lane priority (higher = more urgent)
const SyncLane = 0b0000000000000000000000000000001;
const InputContinuousLane = 0b0000000000000000000000000000100;
const DefaultLane = 0b0000000000000000000000000010000;
const IdleLane = 0b0100000000000000000000000000000;

// User interaction
onClick={() => setState(x)} // InputContinuousLane

// Transition
startTransition(() => setState(x)) // TransitionLane

// useEffect
useEffect(() => setState(x), []) // PassiveLane
```

---

## ✅ 4. Best Practices

### DO ✅

```javascript
// 1. Use concurrent features for better UX
import { useTransition } from "react";

function SearchResults({ query }) {
  const [isPending, startTransition] = useTransition();
  const [results, setResults] = useState([]);

  const handleSearch = (newQuery) => {
    // Mark as non-urgent
    startTransition(() => {
      setResults(expensiveFilter(data, newQuery));
    });
  };

  return (
    <>
      <input onChange={(e) => handleSearch(e.target.value)} />
      {isPending && <Spinner />}
      <ResultsList results={results} />
    </>
  );
}

// 2. Defer expensive computations
import { useDeferredValue } from "react";

function ProductGrid({ searchTerm }) {
  const deferredSearchTerm = useDeferredValue(searchTerm);
  const products = useMemo(
    () => expensiveFilter(allProducts, deferredSearchTerm),
    [deferredSearchTerm],
  );

  return (
    <div className={searchTerm !== deferredSearchTerm ? "opacity-50" : ""}>
      {products.map((p) => (
        <ProductCard key={p.id} product={p} />
      ))}
    </div>
  );
}

// 3. Optimize with keys for efficient reconciliation
function UserList({ users }) {
  return users.map((user) => (
    <UserCard key={user.id} user={user} /> // ✅ Stable key
  ));
}

// 4. Use Suspense for code splitting
const HeavyComponent = lazy(() => import("./HeavyComponent"));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <HeavyComponent />
    </Suspense>
  );
}

// 5. Memoize to avoid unnecessary fiber work
const MemoizedChild = memo(function Child({ data }) {
  // Only re-renders if data changes
  return <div>{data.value}</div>;
});
```

### DON'T ❌

```javascript
// 1. Don't use index as key (breaks reconciliation)
// ❌ BAD
{
  items.map((item, index) => <Item key={index} item={item} />);
}

// ✅ GOOD
{
  items.map((item) => <Item key={item.id} item={item} />);
}

// 2. Don't create components inline
// ❌ BAD: New fiber created on every render
function Parent() {
  return items.map((item) => {
    const Child = () => <div>{item.name}</div>; // New component!
    return <Child key={item.id} />;
  });
}

// ✅ GOOD: Reuse same component
function Parent() {
  return items.map((item) => <Child key={item.id} name={item.name} />);
}

// 3. Don't setState in render
// ❌ BAD: Causes infinite loop
function Component({ value }) {
  const [state, setState] = useState(0);

  if (value > 10) {
    setState(value); // ❌ setState during render!
  }

  return <div>{state}</div>;
}

// ✅ GOOD: Use effect or derived state
function Component({ value }) {
  useEffect(() => {
    if (value > 10) {
      setState(value);
    }
  }, [value]);
}

// 4. Don't block commit phase
// ❌ BAD
useLayoutEffect(() => {
  // Heavy synchronous work blocks paint
  for (let i = 0; i < 1000000; i++) {
    // ...
  }
}, []);

// ✅ GOOD: Use useEffect for non-layout work
useEffect(() => {
  // Runs after paint, doesn't block
  heavyWork();
}, []);
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Misunderstanding Key Prop

```javascript
// ❌ WRONG: Index as key
function TodoList({ todos }) {
  return todos.map((todo, index) => <Todo key={index} todo={todo} />);
}

// Problem: Reordering breaks
// [A, B, C] → [C, A, B]
// Fiber sees: key=0, key=1, key=2 (same keys!)
// Reuses fibers incorrectly, causes bugs

// ✅ CORRECT: Stable unique key
function TodoList({ todos }) {
  return todos.map((todo) => <Todo key={todo.id} todo={todo} />);
}
```

### Mistake #2: Blocking Render Phase

```javascript
// ❌ BLOCKS RENDERING
function HeavyComponent() {
  const data = useMemo(() => {
    // Synchronous heavy work
    let result = [];
    for (let i = 0; i < 10000000; i++) {
      result.push(expensiveCalculation(i));
    }
    return result;
  }, []);

  return <div>{data.length}</div>;
}

// ✅ CHUNKED WITH WEB WORKER OR SUSPENSE
function HeavyComponent() {
  const [data, setData] = useState([]);

  useEffect(() => {
    const worker = new Worker("heavy-worker.js");
    worker.postMessage({ type: "CALCULATE" });
    worker.onmessage = (e) => setData(e.data);

    return () => worker.terminate();
  }, []);

  return <div>{data.length}</div>;
}
```

### Mistake #3: Not Understanding Batching

```javascript
// React 17: Only batches in event handlers
function Component() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  // ❌ In React 17: Two re-renders
  setTimeout(() => {
    setCount((c) => c + 1);
    setFlag((f) => !f);
  }, 1000);

  // React 18: Batched automatically
  // One re-render
}

// React 18: Use flushSync to opt-out of batching
import { flushSync } from "react-dom";

function Component() {
  const handleClick = () => {
    flushSync(() => {
      setCount((c) => c + 1); // Renders immediately
    });
    flushSync(() => {
      setFlag((f) => !f); // Renders immediately
    });
  };
}
```

---

## 🔐 6. Security Considerations

### Denial of Service via Deep Trees

```javascript
// ❌ VULNERABLE: User controls tree depth
function RecursiveComponent({ depth, userControlledDepth }) {
  if (depth >= userControlledDepth) return null;

  return (
    <div>
      <RecursiveComponent
        depth={depth + 1}
        userControlledDepth={userControlledDepth}
      />
    </div>
  );
}

// Attacker sets userControlledDepth=100000 → Stack overflow

// ✅ SAFE: Limit depth
const MAX_DEPTH = 100;

function RecursiveComponent({ depth, userControlledDepth }) {
  const safeDepth = Math.min(userControlledDepth, MAX_DEPTH);
  if (depth >= safeDepth) return null;

  return (
    <div>
      <RecursiveComponent depth={depth + 1} userControlledDepth={safeDepth} />
    </div>
  );
}
```

### XSS via Fiber Internals

```javascript
// ❌ NEVER ACCESS FIBER INTERNALS
const fiber = component._reactInternalFiber;
// fiber.memoizedProps might contain unsanitized data

// ✅ Use React's sanitization
function Component({ userHtml }) {
  // dangerouslySetInnerHTML is the ONLY approved way
  // And should be avoided if possible
  return (
    <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userHtml) }} />
  );
}
```

---

## 🚀 7. Performance Optimization

### 1. Optimize Reconciliation with Keys

```javascript
// ❌ SLOW: No keys, full diff
function List({ items }) {
  return items.map((item) => <Item item={item} />);
}

// ✅ FAST: Keys enable efficient reconciliation
function List({ items }) {
  return items.map((item) => <Item key={item.id} item={item} />);
}
```

### 2. Use Concurrent Features

```javascript
// ✅ Mark expensive updates as transitions
function SearchPage() {
  const [query, setQuery] = useState("");
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    const value = e.target.value;
    setQuery(value); // Urgent: Update input immediately

    startTransition(() => {
      // Non-urgent: Filter results
      setFilteredResults(filterResults(value));
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <Results results={filteredResults} />
    </>
  );
}
```

### 3. Profile with React DevTools

```javascript
// Enable profiler in production
import { Profiler } from "react";

function onRenderCallback(
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime,
) {
  console.log(`${id} (${phase}) took ${actualDuration}ms`);
}

function App() {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <MainContent />
    </Profiler>
  );
}
```

### 4. Avoid Fiber Thrashing

```javascript
// ❌ THRASHING: Rapid state updates
const [value, setValue] = useState(0);

Array(1000)
  .fill(0)
  .forEach((_, i) => {
    setValue(i); // 1000 updates queued!
  });

// ✅ BATCHED: Single update
setValue(999);

// OR use transition
startTransition(() => {
  Array(1000)
    .fill(0)
    .forEach((_, i) => {
      setValue(i); // Batched in transition
    });
});
```

---

## 🧪 8. Code Examples

### Example 1: Understanding Fiber Tree

```javascript
// Component tree
function App() {
  return (
    <div>
      <Header />
      <Main>
        <Sidebar />
        <Content />
      </Main>
    </div>
  );
}

// Fiber tree structure (simplified):
/*
FiberRoot
  └─ current: HostRoot
      └─ child: App
          └─ child: div
              ├─ child: Header
              │   └─ sibling: Main
              │       └─ child: Sidebar
              │           └─ sibling: Content
              
Each fiber has:
- type: The component or DOM tag
- child: First child fiber
- sibling: Next sibling fiber
- return: Parent fiber
- alternate: Corresponding fiber in other tree
*/
```

### Example 2: Priority with useTransition

```javascript
import { useState, useTransition } from "react";

function FilteredList({ data }) {
  const [filter, setFilter] = useState("");
  const [isPending, startTransition] = useTransition();

  const filteredData = useMemo(() => {
    console.log("Filtering...");
    return data.filter((item) => item.name.includes(filter));
  }, [data, filter]);

  const handleChange = (e) => {
    const value = e.target.value;

    // Urgent: Update input immediately (SyncLane)
    setFilter(value);

    // Non-urgent: Filter data (TransitionLane)
    startTransition(() => {
      // This update is deprioritized
      // React can pause it to handle urgent updates
    });
  };

  return (
    <div>
      <input value={filter} onChange={handleChange} placeholder="Filter..." />
      {isPending && <Spinner />}
      <ul className={isPending ? "opacity-50" : ""}>
        {filteredData.map((item) => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Example 3: Suspense and Lazy Loading

```javascript
import { lazy, Suspense } from "react";

// Code-split components
const Dashboard = lazy(() => import("./Dashboard"));
const Reports = lazy(() => import("./Reports"));
const Settings = lazy(() => import("./Settings"));

function App() {
  const [tab, setTab] = useState("dashboard");

  return (
    <div>
      <nav>
        <button onClick={() => setTab("dashboard")}>Dashboard</button>
        <button onClick={() => setTab("reports")}>Reports</button>
        <button onClick={() => setTab("settings")}>Settings</button>
      </nav>

      <Suspense fallback={<PageSpinner />}>
        {tab === "dashboard" && <Dashboard />}
        {tab === "reports" && <Reports />}
        {tab === "settings" && <Settings />}
      </Suspense>
    </div>
  );
}

// Fiber pauses rendering until component loads
// Displays fallback during load
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: Real-Time Dashboard

```javascript
function Dashboard() {
  const [data, setData] = useState([]);
  const [filter, setFilter] = useState("");
  const [isPending, startTransition] = useTransition();

  // Real-time data updates
  useEffect(() => {
    const ws = new WebSocket("wss://data-stream.com");

    ws.onmessage = (event) => {
      const newData = JSON.parse(event.data);

      // Deprioritize data updates
      startTransition(() => {
        setData((prev) => [...prev, newData]);
      });
    };

    return () => ws.close();
  }, []);

  // User interactions remain responsive
  const handleFilter = (e) => {
    setFilter(e.target.value); // High priority
  };

  const filteredData = useMemo(
    () => data.filter((item) => item.name.includes(filter)),
    [data, filter],
  );

  return (
    <div>
      <input value={filter} onChange={handleFilter} />
      {isPending && <div>Updating...</div>}
      <DataGrid data={filteredData} />
    </div>
  );
}
```

### Use Case 2: Large Form with Validation

```javascript
function LargeForm() {
  const [formData, setFormData] = useState({});
  const [errors, setErrors] = useState({});
  const deferredErrors = useDeferredValue(errors);

  const handleChange = (field, value) => {
    // Update input immediately
    setFormData((prev) => ({ ...prev, [field]: value }));

    // Defer expensive validation
    const newErrors = validateForm({ ...formData, [field]: value });
    setErrors(newErrors);
  };

  return (
    <form>
      {fields.map((field) => (
        <div key={field}>
          <input
            value={formData[field] || ""}
            onChange={(e) => handleChange(field, e.target.value)}
          />
          {deferredErrors[field] && (
            <span className="error">{deferredErrors[field]}</span>
          )}
        </div>
      ))}
    </form>
  );
}
```

---

## ❓ 10. Interview Questions

### Q1: What is React Fiber and why was it introduced?

**Answer:**

React Fiber is a complete rewrite of React's reconciliation algorithm (core diffing logic), introduced in React 16.

**Why it was created:**

**Problems with old stack reconciler:**

1. Synchronous rendering blocked main thread
2. No way to pause or abort work
3. Couldn't prioritize updates (user input vs data fetch)
4. Large trees caused frame drops (<60fps)

**Fiber's solutions:**

1. **Incremental rendering** - Split work into chunks
2. **Prioritization** - Urgent updates (clicks) > non-urgent (data)
3. **Interruptible** - Pause work, yield to browser, resume
4. **Concurrent mode** - Multiple versions of UI in memory

**Real impact:**

- Smooth animations during heavy renders
- Responsive inputs in complex UIs
- Better perceived performance

**Follow-up:** How does Fiber achieve this?

- Uses **time-slicing**: breaks work into 5ms chunks
- Maintains **two fiber trees**: current (displayed) and workInProgress (computing)
- Implements **priority queue** for updates (lanes)

---

### Q2: Explain the render and commit phases in Fiber.

**Answer:**

**Render Phase (Async, Interruptible):**

- Computes what changed
- Creates new fiber tree (workInProgress)
- Calls render methods, hooks
- Can be paused/resumed/aborted
- No side effects allowed
- May be called multiple times

```javascript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress); // Can pause here
  }
}
```

**Commit Phase (Sync, Non-Interruptible):**

- Applies changes to DOM
- Runs lifecycle methods (componentDidMount, useLayoutEffect)
- Must complete atomically (can't pause)
- Side effects executed

```javascript
function commitRoot() {
  commitBeforeMutationEffects(); // getSnapshotBeforeUpdate
  commitMutationEffects(); // DOM updates
  commitLayoutEffects(); // useLayoutEffect
  schedulePassiveEffects(); // useEffect (async)
}
```

**Key difference:**

- Render = "What should change?" (pure, can retry)
- Commit = "Make it change" (side effects, must succeed)

---

### Q3: What are React lanes and how do they work?

**Answer:**

**Lanes** are Fiber's priority system for scheduling updates. Think of them as highway lanes - urgent traffic gets fast lanes.

**Lane types (simplified):**

```javascript
SyncLane; // Immediate (discrete user input)
InputContinuousLane; // Continuous input (scroll, drag)
DefaultLane; // Normal updates
TransitionLane; // Marked with startTransition
IdleLane; // Low priority (analytics)
```

**Example:**

```javascript
// User clicks button → SyncLane (highest priority)
<button onClick={() => setCount((c) => c + 1)}>Click</button>;

// Wrapped in transition → TransitionLane (lower priority)
startTransition(() => {
  setResults(filterData(query)); // Can be interrupted
});

// Effect → PassiveLane (runs after paint)
useEffect(() => {
  logAnalytics(); // Doesn't block UI
}, []);
```

**How it works:**

1. Each update assigned a lane based on source
2. Fiber processes highest priority lanes first
3. Lower priority work can be interrupted by higher priority
4. Lanes can be batched if same priority

**Real-world impact:**

- Click handlers never feel laggy
- Heavy computations don't block interactions
- Smooth UX in complex apps

---

### Q4: How does the key prop help Fiber?

**Answer:**

**Keys help Fiber identify which elements changed, added, or removed** during reconciliation.

**Without keys:**

```javascript
// Before: [A, B, C]
// After:  [C, A, B]

// Fiber without keys:
// - Sees 3 elements → 3 elements
// - Updates element 0: A → C (wrong!)
// - Updates element 1: B → A (wrong!)
// - Updates element 2: C → B (wrong!)
// Result: 3 updates, all wrong
```

**With keys:**

```javascript
// Before: [<A key="a" />, <B key="b" />, <C key="c" />]
// After:  [<C key="c" />, <A key="a" />, <B key="b" />]

// Fiber with keys:
// - Sees key="c" moved to position 0
// - Sees key="a", key="b" shifted
// - Reuses existing fibers, just reorders
// Result: 0 updates, just reorder
```

**Why index as key is bad:**

```javascript
// ❌ Index as key
{
  todos.map((todo, i) => <Todo key={i} {...todo} />);
}

// Problem: Keys always 0, 1, 2...
// Delete first todo:
// Before: [<Todo key={0} id="a" />, <Todo key={1} id="b" />]
// After:  [<Todo key={0} id="b" />]
// Fiber reuses key={0} fiber, updates id "a"→"b" (wrong!)
// Loses internal state (input values, etc.)
```

**✅ Best practices:**

- Use stable, unique IDs
- Don't use index (unless static list)
- Don't generate keys on render (changes every time)

---

### Q5: What is the difference between useEffect and useLayoutEffect in terms of Fiber?

**Answer:**

**useLayoutEffect:**

- Runs synchronously **during commit phase**
- Fires **before browser paints**
- Blocks visual updates
- Use for DOM measurements, preventing flicker

```javascript
useLayoutEffect(() => {
  // Runs BEFORE paint
  const height = ref.current.offsetHeight;
  setHeight(height); // Blocks paint until complete
}, []);
```

**useEffect:**

- Runs **after commit phase** (asynchronously)
- Fires **after browser paints**
- Doesn't block visual updates
- Use for data fetching, subscriptions, logging

```javascript
useEffect(() => {
  // Runs AFTER paint
  fetchData().then(setData); // Doesn't block rendering
}, []);
```

**Timeline:**

```
1. Render phase (compute changes)
2. Commit phase BEGIN
   ├─ commitMutationEffects (DOM updates)
   ├─ commitLayoutEffects (useLayoutEffect runs HERE)
   └─ commitRootCompleted
3. Browser paints screen
4. Passive effects (useEffect runs HERE)
```

**When to use which:**

| Scenario         | Hook              | Why                            |
| ---------------- | ----------------- | ------------------------------ |
| Data fetching    | `useEffect`       | No need to block paint         |
| Measure DOM      | `useLayoutEffect` | Need size before paint         |
| Subscriptions    | `useEffect`       | Async is fine                  |
| Prevent flicker  | `useLayoutEffect` | Must update before visible     |
| Animations       | `useEffect`       | requestAnimationFrame is async |
| Focus management | `useLayoutEffect` | Must happen before user sees   |

---

## 🧩 11. Design Patterns

### Pattern 1: Optimistic Updates with Transitions

```javascript
function TodoApp() {
  const [todos, setTodos] = useState([]);
  const [isPending, startTransition] = useTransition();

  const addTodo = async (text) => {
    const tempId = Date.now();
    const optimisticTodo = { id: tempId, text, synced: false };

    // Immediate optimistic update (SyncLane)
    setTodos((prev) => [...prev, optimisticTodo]);

    try {
      const newTodo = await api.createTodo(text);

      // Real update in transition (TransitionLane)
      startTransition(() => {
        setTodos((prev) => prev.map((t) => (t.id === tempId ? newTodo : t)));
      });
    } catch (error) {
      // Rollback on failure
      setTodos((prev) => prev.filter((t) => t.id !== tempId));
    }
  };

  return (
    <div>
      <TodoForm onSubmit={addTodo} />
      <TodoList todos={todos} isPending={isPending} />
    </div>
  );
}
```

### Pattern 2: Deferred Value for Expensive Filtering

```javascript
function ProductCatalog({ products }) {
  const [search, setSearch] = useState("");
  const deferredSearch = useDeferredValue(search);

  // Expensive filtering uses deferred value
  const filtered = useMemo(() => {
    console.log("Filtering with:", deferredSearch);
    return products.filter((p) =>
      p.name.toLowerCase().includes(deferredSearch.toLowerCase()),
    );
  }, [products, deferredSearch]);

  return (
    <div>
      <input
        value={search}
        onChange={(e) => setSearch(e.target.value)}
        placeholder="Search products..."
      />
      <div className={search !== deferredSearch ? "opacity-50" : ""}>
        {filtered.map((p) => (
          <ProductCard key={p.id} product={p} />
        ))}
      </div>
    </div>
  );
}
```

---

## 🎯 Summary

React Fiber is the heart of modern React:

- **Architecture:** Linked list of fiber nodes (not stack)
- **Goal:** Interruptible, prioritized rendering
- **Phases:** Render (async) → Commit (sync)
- **Priority:** Lanes system for update scheduling
- **Features:** Time-slicing, Suspense, Concurrent Mode

**Production benefits:**

- Smooth UIs (no janky animations)
- Responsive inputs (even during heavy renders)
- Better perceived performance
- Enables Concurrent features (transitions, deferred values)

**Interview essentials:**

- Explain why Fiber was created (old reconciler limitations)
- Understand render vs commit phases
- Know priority system (lanes)
- Can discuss concurrent features
- Relate to real-world performance issues

Remember: Fiber is an **implementation detail**, but understanding it makes you a better React engineer who can diagnose performance issues and write optimized code.
