# Render and Commit Phases

## Core Concept

React's work is divided into two phases: **Render** (interruptible, pure computation) and **Commit** (synchronous, DOM mutations). Understanding this separation is key to React's concurrent features.

---

## Two-Phase Architecture

```typescript
// Phase 1: RENDER (Interruptible)
// - Compute new virtual DOM
// - Call render functions
// - Perform reconciliation
// - Pure computation (no side effects)
// - Can be paused/resumed

// Phase 2: COMMIT (Synchronous)
// - Apply changes to DOM
// - Run layout effects
// - Update refs
// - Side effects happen here
// - Must complete synchronously
```

---

## Render Phase

```typescript
function Component() {
  console.log('Rendering...'); // ⚠️ May log multiple times!

  const [count, setCount] = useState(0);

  // Render phase work:
  // - Execute function body
  // - Create React elements
  // - Compute diffs

  return <div>{count}</div>;
}

// In concurrent mode, render may be:
// - Started
// - Paused (higher priority work comes in)
// - Discarded (props changed)
// - Resumed
// - Completed
```

---

## Commit Phase

```typescript
function Component() {
  const [count, setCount] = useState(0);

  // Commit phase work:
  useLayoutEffect(() => {
    console.log('Layout effect'); // Runs synchronously during commit
    // DOM mutations are flushed but not painted yet
  });

  useEffect(() => {
    console.log('Effect'); // Runs after commit + paint
    // DOM is painted, user sees update
  });

  return <div ref={divRef}>{count}</div>;
  // Ref updated during commit phase
}
```

---

## Execution Order

```typescript
function Parent() {
  console.log('1. Parent render');

  useLayoutEffect(() => {
    console.log('4. Parent layout effect');
  });

  useEffect(() => {
    console.log('6. Parent effect');
  });

  return <Child />;
}

function Child() {
  console.log('2. Child render');

  useLayoutEffect(() => {
    console.log('3. Child layout effect');
  });

  useEffect(() => {
    console.log('5. Child effect');
  });

  return <div>Child</div>;
}

// Output:
// 1. Parent render
// 2. Child render
// 3. Child layout effect (children first)
// 4. Parent layout effect
// 5. Child effect (children first)
// 6. Parent effect
```

---

## Render Phase Purity

```typescript
// ❌ BAD - Side effects in render
function BadComponent() {
  // These run during render phase (interruptible)
  fetch('/api/data');           // ❌ Network request
  localStorage.setItem('x', 'y'); // ❌ Mutation
  console.log('Rendering');      // ⚠️ May log multiple times

  return <div>Bad</div>;
}

// ✅ GOOD - Side effects in effects
function GoodComponent() {
  useEffect(() => {
    // These run during commit phase (once)
    fetch('/api/data');           // ✅ In effect
    localStorage.setItem('x', 'y'); // ✅ In effect
  }, []);

  return <div>Good</div>;
}
```

---

## Fiber Work Loop

```typescript
// Simplified React work loop
function workLoop(deadline: IdleDeadline) {
  let shouldYield = false;

  // Render phase - interruptible
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }

  // Commit phase - synchronous
  if (!nextUnitOfWork && wipRoot) {
    commitRoot();
  }

  requestIdleCallback(workLoop);
}

function performUnitOfWork(fiber: Fiber) {
  // 1. Reconcile - compare with previous fiber
  reconcileChildren(fiber);

  // 2. Return next unit of work
  if (fiber.child) return fiber.child;
  if (fiber.sibling) return fiber.sibling;
  return fiber.parent?.sibling;
}

function commitRoot() {
  // Commit phase - must complete synchronously
  commitWork(wipRoot.child);
  currentRoot = wipRoot;
  wipRoot = null;
}
```

---

## Concurrent Rendering

```typescript
function App() {
  const [count, setCount] = useState(0);
  const [search, setSearch] = useState('');

  // High priority - immediate
  const handleClick = () => {
    setCount(count + 1);
  };

  // Low priority - can be interrupted
  const handleSearch = (e: React.ChangeEvent<HTMLInputElement>) => {
    startTransition(() => {
      setSearch(e.target.value);
    });
  };

  return (
    <>
      <button onClick={handleClick}>Count: {count}</button>
      <input onChange={handleSearch} />
      <SlowList search={search} />
    </>
  );
}

// If user clicks button while SlowList is rendering:
// 1. React pauses SlowList render
// 2. Renders button update (high priority)
// 3. Commits button update
// 4. Resumes SlowList render
```

---

## useLayoutEffect vs useEffect

```typescript
function LayoutExample() {
  const [width, setWidth] = useState(0);
  const divRef = useRef<HTMLDivElement>(null);

  // useLayoutEffect - before paint
  useLayoutEffect(() => {
    const measured = divRef.current?.offsetWidth || 0;
    setWidth(measured);
    // DOM updated but not painted yet
    // No visual flicker
  }, []);

  // useEffect - after paint
  useEffect(() => {
    const measured = divRef.current?.offsetWidth || 0;
    setWidth(measured);
    // DOM already painted
    // May cause visual flicker
  }, []);

  return <div ref={divRef}>Width: {width}</div>;
}

// Timeline:
// 1. Render phase - create virtual DOM
// 2. Commit phase - update DOM
// 3. useLayoutEffect - before browser paint
// 4. Browser paint - user sees update
// 5. useEffect - after paint
```

---

## Batching Updates

```typescript
function Component() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  // React 18: Automatic batching
  const handleClick = () => {
    setCount(c => c + 1);  // Not rendered yet
    setFlag(f => !f);       // Not rendered yet
    // Both updates batched → single render
  };

  // Even in async code
  const handleAsync = async () => {
    await fetch('/api/data');
    setCount(c => c + 1);  // Batched
    setFlag(f => !f);       // Batched
    // Single render after fetch
  };

  // Opt out with flushSync
  const handleFlush = () => {
    flushSync(() => {
      setCount(c => c + 1); // Rendered immediately
    });
    setFlag(f => !f);        // Rendered separately
    // Two separate renders
  };

  return <div>{count} {flag ? 'true' : 'false'}</div>;
}
```

---

## Passive vs Active Effects

```typescript
function Component() {
  // Passive effect (useEffect)
  // - Runs after paint
  // - Non-blocking
  // - Can be deferred
  useEffect(() => {
    console.log("Passive effect");
    fetch("/api/track-view");
  }, []);

  // Active effect (useLayoutEffect)
  // - Runs before paint
  // - Blocking
  // - Cannot be deferred
  useLayoutEffect(() => {
    console.log("Active effect");
    const height = element.offsetHeight;
    element.style.height = `${height}px`;
  }, []);
}
```

---

## Commit Phase Timeline

```typescript
// 1. Before mutation
// - getSnapshotBeforeUpdate (class components)

// 2. Mutation phase
// - DOM mutations applied
// - Refs updated

function Component() {
  const divRef = useRef<HTMLDivElement>(null);

  // Ref updated here ↓
  return <div ref={divRef}>Content</div>;
}

// 3. Layout effects
// - useLayoutEffect callbacks
// - componentDidMount/componentDidUpdate

// 4. Browser paint
// - User sees update

// 5. Passive effects
// - useEffect callbacks
```

---

## Real-World: Measuring DOM

```typescript
function MeasuredBox() {
  const [dimensions, setDimensions] = useState({ width: 0, height: 0 });
  const boxRef = useRef<HTMLDivElement>(null);

  // ❌ BAD - useEffect causes flicker
  useEffect(() => {
    if (boxRef.current) {
      setDimensions({
        width: boxRef.current.offsetWidth,
        height: boxRef.current.offsetHeight
      });
      // Timeline:
      // 1. Initial render with 0x0
      // 2. Browser paints 0x0 (user sees)
      // 3. Effect measures and updates
      // 4. Re-render with real dimensions
      // 5. Browser paints (flicker visible)
    }
  }, []);

  // ✅ GOOD - useLayoutEffect prevents flicker
  useLayoutEffect(() => {
    if (boxRef.current) {
      setDimensions({
        width: boxRef.current.offsetWidth,
        height: boxRef.current.offsetHeight
      });
      // Timeline:
      // 1. Initial render with 0x0
      // 2. Layout effect measures and updates
      // 3. Re-render with real dimensions
      // 4. Browser paints (no flicker)
    }
  }, []);

  return (
    <div ref={boxRef}>
      Size: {dimensions.width}x{dimensions.height}
    </div>
  );
}
```

---

## Avoiding Blocking

```typescript
function ExpensiveComponent({ data }: { data: Item[] }) {
  // ❌ BAD - Expensive computation during render
  const sorted = [...data].sort((a, b) => {
    // Complex sorting logic
    return expensiveComparison(a, b);
  });

  // Blocks render phase for all components
  return <List items={sorted} />;
}

// ✅ GOOD - Defer with useDeferredValue
function BetterComponent({ data }: { data: Item[] }) {
  const deferredData = useDeferredValue(data);

  const sorted = useMemo(() => {
    return [...deferredData].sort((a, b) => {
      return expensiveComparison(a, b);
    });
  }, [deferredData]);

  return <List items={sorted} />;
}
```

---

## Best Practices

✅ **Keep render phase pure** - no side effects  
✅ **Use useEffect** for async work  
✅ **Use useLayoutEffect** for DOM measurements  
✅ **Let React batch updates** - trust automatic batching  
✅ **Use useDeferredValue** for expensive computations  
❌ **Don't mutate state directly** during render  
❌ **Don't rely on render phase count** - may render multiple times  
❌ **Don't block with useLayoutEffect** - use sparingly

---

## Key Takeaways

1. **Render phase is interruptible** and pure
2. **Commit phase is synchronous** with side effects
3. **useLayoutEffect runs before paint** (blocking)
4. **useEffect runs after paint** (non-blocking)
5. **React batches updates** automatically in React 18
6. **Concurrent mode can pause/resume** render work
7. **Keep render functions pure** for predictable behavior
