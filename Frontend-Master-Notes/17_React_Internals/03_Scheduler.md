# React Scheduler

## Core Concept

The React Scheduler is a **cooperative multitasking** system that enables React to:

- Break rendering work into chunks
- Prioritize urgent updates (user input) over less urgent ones (data fetching)
- Yield to the browser to keep UI responsive
- Resume work when idle

---

## Why Scheduler Exists

### **Problem: Blocking Render**

```javascript
// Long synchronous render blocks everything
function ExpensiveComponent() {
  const items = Array(10000).fill(0);
  return (
    <div>
      {items.map((_, i) => (
        <div key={i}>Item {i}</div>
      ))}
    </div>
  );
}
// User can't interact until render completes!
```

### **Solution: Time Slicing**

Break work into chunks, allowing browser to handle user input between chunks.

---

## Scheduler Priorities

React Scheduler has **5 priority levels**:

```typescript
enum Priority {
  ImmediatePriority = 1, // Sync (e.g., user input, errors)
  UserBlockingPriority = 2, // User interactions (clicks, typing)
  NormalPriority = 3, // Default (data fetching, updates)
  LowPriority = 4, // Can wait (analytics, logging)
  IdlePriority = 5, // Run when idle (pre-fetch, caching)
}
```

---

## How It Works

### **1. Task Queue**

```javascript
// Simplified scheduler concept
class Scheduler {
  taskQueue = [];

  scheduleTask(callback, priority) {
    this.taskQueue.push({ callback, priority });
    this.taskQueue.sort((a, b) => a.priority - b.priority);
  }

  workLoop(deadline) {
    while (this.taskQueue.length > 0 && deadline.timeRemaining() > 0) {
      const task = this.taskQueue.shift();
      task.callback();
    }

    if (this.taskQueue.length > 0) {
      requestIdleCallback(this.workLoop);
    }
  }
}
```

### **2. Time Slicing**

```javascript
const FRAME_BUDGET = 5; // 5ms per frame (to hit 60fps)

function workLoop(deadline) {
  let shouldYield = false;

  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }

  if (nextUnitOfWork) {
    requestIdleCallback(workLoop); // Continue later
  } else {
    commitRoot(); // Finished, commit to DOM
  }
}

requestIdleCallback(workLoop);
```

---

## Priority Expiration

Tasks have expiration times based on priority:

```javascript
const IMMEDIATE_PRIORITY_TIMEOUT = -1; // Never expires
const USER_BLOCKING_PRIORITY_TIMEOUT = 250; // 250ms
const NORMAL_PRIORITY_TIMEOUT = 5000; // 5s
const LOW_PRIORITY_TIMEOUT = 10000; // 10s
const IDLE_PRIORITY_TIMEOUT = maxInt; // Infinity

function getExpirationTime(priority) {
  const currentTime = Date.now();
  switch (priority) {
    case ImmediatePriority:
      return currentTime + IMMEDIATE_PRIORITY_TIMEOUT;
    case UserBlockingPriority:
      return currentTime + USER_BLOCKING_PRIORITY_TIMEOUT;
    // ... other cases
  }
}
```

---

## React 18: startTransition

Mark updates as non-urgent:

```typescript
import { startTransition } from 'react';

function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const handleChange = (e) => {
    // Urgent: Update input immediately
    setQuery(e.target.value);

    // Non-urgent: Can be interrupted
    startTransition(() => {
      const filtered = heavyFilter(data, e.target.value);
      setResults(filtered);
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      <ResultsList results={results} />
    </>
  );
}
```

---

## useTransition Hook

```typescript
function App() {
  const [isPending, startTransition] = useTransition();
  const [tab, setTab] = useState('about');

  const selectTab = (nextTab) => {
    startTransition(() => {
      setTab(nextTab);
    });
  };

  return (
    <>
      <button onClick={() => selectTab('about')}>About</button>
      <button onClick={() => selectTab('posts')}>Posts</button>
      {isPending && <Spinner />}
      <TabContent tab={tab} />
    </>
  );
}
```

---

## useDeferredValue

Defer re-rendering of expensive components:

```typescript
function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {/* Uses old value while new one renders */}
      <ExpensiveList query={deferredQuery} />
    </>
  );
}
```

---

## Scheduler API (Internal)

React uses these internally:

```typescript
import * as Scheduler from "scheduler";

// Schedule callback at priority
Scheduler.scheduleCallback(Scheduler.ImmediatePriority, () => {
  // Urgent work
});

// Cancel scheduled work
const task = Scheduler.scheduleCallback(priority, callback);
Scheduler.cancelCallback(task);

// Check if should yield
if (Scheduler.shouldYield()) {
  // Pause and resume later
  return;
}
```

---

## MessageChannel for Scheduling

React uses MessageChannel (not setTimeout) for better control:

```javascript
const channel = new MessageChannel();
const port = channel.port2;

channel.port1.onmessage = () => {
  // Process work
  performWork();
};

function scheduleWork() {
  port.postMessage(null); // Triggers async work
}
```

---

## Real-World Example: Search

```typescript
function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (value: string) => {
    // High priority - update input immediately
    setQuery(value);

    // Low priority - can be interrupted
    startTransition(() => {
      const newResults = expensiveSearch(value);
      setResults(newResults);
    });
  };

  return (
    <div>
      <input
        value={query}
        onChange={e => handleSearch(e.target.value)}
      />
      {isPending ? <Spinner /> : <Results data={results} />}
    </div>
  );
}
```

---

## Performance Implications

### **Without Scheduler (React <18)**

```javascript
// All updates are same priority
setState(newState); // Blocks UI until complete
```

### **With Scheduler (React >=18)**

```javascript
// Urgent update
setState(newState);

// Non-urgent update
startTransition(() => {
  setState(expensiveState); // Can be interrupted
});
```

---

## Debugging Scheduler

```typescript
// React DevTools Profiler shows:
// - Which updates were transitions
// - How long renders took
// - When renders were interrupted

// Enable Concurrent Features
import { unstable_trace as trace } from "scheduler/tracing";

trace("user action", performance.now(), () => {
  // Traced work
});
```

---

## Best Practices

✅ **Use startTransition** for non-urgent updates  
✅ **Mark heavy renders** as transitions  
✅ **Show pending state** with isPending  
✅ **Use useDeferredValue** for search/filter  
✅ **Prioritize user input** over data updates  
❌ **Don't overuse transitions** - only for heavy work  
❌ **Don't wrap all updates** - use for expensive renders

---

## Key Takeaways

1. **Scheduler enables time slicing** - breaks work into chunks
2. **5 priority levels** - from immediate to idle
3. **startTransition marks non-urgent** updates
4. **useTransition provides pending state**
5. **useDeferredValue defers expensive re-renders**
6. **Scheduler yields to browser** to keep UI responsive
7. **React 18 concurrent features** rely on Scheduler
