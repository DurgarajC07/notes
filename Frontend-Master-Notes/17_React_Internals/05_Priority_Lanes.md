# Priority Lanes

## Core Concept

React 18's concurrent rendering uses a **lanes-based priority system** to schedule and interrupt work. Each update is assigned a lane (priority level) that determines when it should be processed.

---

## Lane System Overview

```typescript
// React's lane priorities (simplified)
const SyncLane = 0b0000000000000000000000000000001;
const InputContinuousLane = 0b0000000000000000000000000000100;
const DefaultLane = 0b0000000000000000000000000010000;
const TransitionLane = 0b0000000000000000000001000000000;
const IdleLane = 0b0100000000000000000000000000000;

// Multiple lanes can be combined with bitwise OR
const lanes = SyncLane | DefaultLane;
```

---

## Priority Levels

### **1. Sync (Highest Priority)**

```typescript
// Immediate updates that must complete synchronously
function SyncExample() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    flushSync(() => {
      setCount(count + 1);
      // Rendered immediately, blocks everything
    });
  };

  return <button onClick={handleClick}>Count: {count}</button>;
}

// Use cases:
// - Controlled inputs
// - Critical UI feedback
// - Error boundaries
```

### **2. Input Continuous**

```typescript
// High priority but can be batched
function InputExample() {
  const [value, setValue] = useState('');

  // Input events are InputContinuous priority
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setValue(e.target.value);
    // High priority, but can batch multiple keystrokes
  };

  return <input value={value} onChange={handleChange} />;
}
```

### **3. Default Priority**

```typescript
// Normal updates
function DefaultExample() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    // Effect updates are default priority
    const timer = setTimeout(() => {
      setCount(count + 1);
    }, 1000);

    return () => clearTimeout(timer);
  }, [count]);

  return <div>{count}</div>;
}
```

### **4. Transition Priority**

```typescript
// Low priority, interruptible updates
import { useTransition } from 'react';

function TransitionExample() {
  const [isPending, startTransition] = useTransition();
  const [input, setInput] = useState('');
  const [list, setList] = useState<string[]>([]);

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;

    // High priority - immediate
    setInput(value);

    // Low priority - interruptible
    startTransition(() => {
      setList(generateHugeList(value));
    });
  };

  return (
    <>
      <input value={input} onChange={handleChange} />
      {isPending && <Spinner />}
      <List items={list} />
    </>
  );
}
```

### **5. Idle Priority**

```typescript
// Lowest priority - run when browser is idle
function IdleExample() {
  const [heavyData, setHeavyData] = useState(null);

  useEffect(() => {
    // Schedule for idle time
    const handle = requestIdleCallback(() => {
      const data = processHeavyComputation();
      setHeavyData(data);
    });

    return () => cancelIdleCallback(handle);
  }, []);

  return <div>{heavyData || 'Loading...'}</div>;
}
```

---

## Lane Selection

```typescript
// React determines lane based on context:

// 1. Event type
const onClick = () => {
  setState(...); // DiscreteEventLane (sync)
};

const onMouseMove = () => {
  setState(...); // InputContinuousLane
};

// 2. Transition context
const handleSearch = () => {
  startTransition(() => {
    setState(...); // TransitionLane
  });
};

// 3. Deferred value
const deferredValue = useDeferredValue(value);
// Updates to deferredValue use TransitionLane

// 4. Current render lane
// Updates during render inherit current lane
```

---

## Lane Merging

```typescript
function Component() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  const handleClick = () => {
    // Multiple updates in same event
    setCount(count + 1);      // DiscreteEventLane
    setText('Updated');        // Same lane
    // Both updates batched in same lane
  };

  const handleTransition = () => {
    setCount(count + 1);       // DiscreteEventLane

    startTransition(() => {
      setText('Updated');      // TransitionLane
      // Two separate lanes - rendered separately
    });
  };

  return (
    <>
      <button onClick={handleClick}>Click</button>
      <button onClick={handleTransition}>Transition</button>
    </>
  );
}
```

---

## Interrupting Low Priority Work

```typescript
function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Result[]>([]);

  const handleSearch = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;

    // Immediate - high priority
    setQuery(value);

    // Deferred - low priority
    startTransition(() => {
      const filtered = expensiveFilter(data, value);
      setResults(filtered);
    });
  };

  return (
    <>
      {/* User types 'react' quickly: r-e-a-c-t

          Timeline:
          1. User types 'r' (high priority)
             - setQuery('r') renders immediately
             - startTransition begins filtering 'r'

          2. User types 'e' before filtering completes (high priority)
             - setQuery('e') interrupts transition
             - Filtering 'r' is discarded
             - startTransition begins filtering 're'

          3. User types 'a' (high priority)
             - setQuery('a') interrupts transition
             - Filtering 're' is discarded
             - startTransition begins filtering 'rea'

          ... and so on

          Result: Only final filter completes!
      */}
      <input value={query} onChange={handleSearch} />
      <Results items={results} />
    </>
  );
}
```

---

## useDeferredValue

```typescript
import { useDeferredValue } from 'react';

function DeferredExample({ value }: { value: string }) {
  // Deferred version lags behind during updates
  const deferredValue = useDeferredValue(value);

  return (
    <>
      {/* Renders immediately with latest value */}
      <input value={value} />

      {/* Renders with deferred value (low priority) */}
      <SlowList query={deferredValue} />
    </>
  );
}

// How it works:
// 1. value changes from 'a' to 'ab'
// 2. Input updates immediately (high priority)
// 3. SlowList still shows 'a' (old deferred value)
// 4. If value changes to 'abc', SlowList update for 'ab' is skipped
// 5. SlowList eventually updates to 'abc' when idle
```

---

## Suspense and Transitions

```typescript
import { Suspense, lazy } from 'react';

const HeavyComponent = lazy(() => import('./HeavyComponent'));

function SuspenseExample() {
  const [show, setShow] = useState(false);
  const [isPending, startTransition] = useTransition();

  const handleShow = () => {
    // Without transition:
    // setShow(true); // Immediate, shows fallback spinner

    // With transition:
    startTransition(() => {
      setShow(true);
      // Keeps old UI visible until new UI is ready
      // No spinner flash
    });
  };

  return (
    <>
      <button onClick={handleShow}>
        Show {isPending && <Spinner />}
      </button>

      <Suspense fallback={<Loading />}>
        {show && <HeavyComponent />}
      </Suspense>
    </>
  );
}
```

---

## Priority Inversion Prevention

```typescript
function PriorityExample() {
  const [highPriority, setHighPriority] = useState(0);
  const [lowPriority, setLowPriority] = useState(0);

  const handleClick = () => {
    // High priority update
    setHighPriority(h => h + 1);

    // If low priority was rendering when click happens:
    // 1. Low priority work is interrupted
    // 2. High priority work starts immediately
    // 3. High priority completes and commits
    // 4. Low priority resumes from scratch (not paused)

    startTransition(() => {
      setLowPriority(l => l + 1);
    });
  };

  return (
    <>
      <div>High: {highPriority}</div>
      <div>Low: {lowPriority}</div>
      <button onClick={handleClick}>Update</button>
    </>
  );
}
```

---

## Hydration Priority

```typescript
// SSR hydration uses special lanes
function ServerRendered() {
  const [count, setCount] = useState(0);

  // During hydration:
  // 1. Urgent interactions (clicks) use sync lane
  // 2. Regular hydration uses default lane
  // 3. Transitions during hydration use transition lane

  const handleClick = () => {
    setCount(count + 1);
    // If clicked before hydration completes:
    // - Click is processed immediately (sync lane)
    // - Hydration continues afterwards
  };

  return <button onClick={handleClick}>Count: {count}</button>;
}
```

---

## Lane Debugging

```typescript
// Enable React DevTools profiler to see lanes
function DebugLanes() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    // Log when component renders
    console.log('Rendered at:', performance.now());
  });

  const handleSync = () => {
    flushSync(() => {
      setCount(count + 1);
      console.log('Sync update');
    });
  };

  const handleTransition = () => {
    startTransition(() => {
      setCount(count + 1);
      console.log('Transition update');
    });
  };

  return (
    <>
      <button onClick={handleSync}>Sync</button>
      <button onClick={handleTransition}>Transition</button>
      <div>Count: {count}</div>
    </>
  );
}
```

---

## Real-World: Search with Autocomplete

```typescript
import { useState, useDeferredValue, useTransition } from 'react';

function SearchAutocomplete() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  const deferredQuery = useDeferredValue(query);

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    // Immediate - sync lane
    setQuery(e.target.value);
  };

  // Results use deferred value - transition lane
  const results = useMemo(() => {
    return searchDatabase(deferredQuery);
  }, [deferredQuery]);

  return (
    <div>
      <input
        value={query}
        onChange={handleChange}
        placeholder="Search..."
      />

      {/* Input always responsive */}
      {/* Results may lag during fast typing */}

      {isPending && <Spinner />}
      <Results items={results} />
    </div>
  );
}
```

---

## Best Practices

✅ **Use startTransition** for non-urgent updates  
✅ **Use useDeferredValue** for expensive derived state  
✅ **Let React manage priorities** - trust the system  
✅ **Mark UI feedback as urgent** - clicks, inputs  
✅ **Mark data fetching as transition** when appropriate  
❌ **Don't overuse flushSync** - breaks concurrency  
❌ **Don't mark everything as transition** - loses benefits  
❌ **Don't fight the priority system** - work with it

---

## Key Takeaways

1. **Lanes are priority levels** for scheduling work
2. **High priority work interrupts** low priority work
3. **startTransition marks updates** as interruptible
4. **useDeferredValue creates** lagging state value
5. **React automatically assigns** lanes based on context
6. **Transitions prevent UI blocking** during expensive updates
7. **Priority system enables** responsive concurrent UI
