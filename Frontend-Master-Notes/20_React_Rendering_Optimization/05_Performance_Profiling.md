# React Performance Profiling

## Core Concept

React DevTools Profiler measures component render performance to identify bottlenecks and optimize React applications.

---

## Profiler API

Programmatic performance tracking:

```typescript
import { Profiler, ProfilerOnRenderCallback } from 'react';

function App() {
  const onRender: ProfilerOnRenderCallback = (
    id,                  // Component ID
    phase,               // "mount" or "update"
    actualDuration,      // Time spent rendering
    baseDuration,        // Estimated time without memoization
    startTime,           // When render started
    commitTime,          // When committed to DOM
    interactions         // Set of interactions (deprecated)
  ) => {
    console.log(`${id} (${phase}): ${actualDuration}ms`);

    // Send to analytics
    if (actualDuration > 16) { // Slower than 60fps
      analytics.track('slow-render', {
        component: id,
        duration: actualDuration
      });
    }
  };

  return (
    <Profiler id="App" onRender={onRender}>
      <YourComponents />
    </Profiler>
  );
}
```

---

## React DevTools Profiler

### **Recording a Profile**

1. Open React DevTools
2. Click "Profiler" tab
3. Click record button (⚫)
4. Interact with your app
5. Stop recording

### **Reading the Flamegraph**

```
App (12.5ms)
├── Header (0.3ms)
├── Sidebar (1.2ms)
└── Content (11.0ms)      ← Expensive!
    ├── List (10.5ms)     ← Very expensive!
    └── Footer (0.5ms)
```

**Colors indicate render time:**

- 🟩 Green = Fast (<5ms)
- 🟨 Yellow = Medium (5-15ms)
- 🟧 Orange = Slow (15-30ms)
- 🟥 Red = Very slow (>30ms)

---

## Ranked Chart

Shows components by render time:

```
1. List          10.5ms  ← Optimize first
2. Content       11.0ms
3. Sidebar        1.2ms
4. Header         0.3ms
5. Footer         0.5ms
```

---

## Interactions Tracking (Deprecated)

Use `useTransition` instead:

```typescript
import { useTransition } from 'react';

function SearchBox() {
  const [isPending, startTransition] = useTransition();
  const [query, setQuery] = useState('');

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setQuery(value);

    // Mark expensive update as transition
    startTransition(() => {
      setSearchResults(filterResults(value));
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
    </>
  );
}
```

---

## Performance Marks

Custom timing with Performance API:

```typescript
import { useEffect } from 'react';

function ExpensiveComponent() {
  useEffect(() => {
    performance.mark('component-mount-start');

    // Do expensive work
    heavyComputation();

    performance.mark('component-mount-end');
    performance.measure(
      'component-mount',
      'component-mount-start',
      'component-mount-end'
    );

    const measure = performance.getEntriesByName('component-mount')[0];
    console.log(`Mounted in ${measure.duration}ms`);
  }, []);

  return <div>Content</div>;
}
```

---

## Real-World Example: Slow List

```typescript
// ❌ BAD - re-renders all items
function BadList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(item => (
        <ListItem key={item.id} item={item} />
      ))}
    </ul>
  );
}

function ListItem({ item }: { item: Item }) {
  console.log('Rendering ListItem:', item.id);
  return <li>{item.name}</li>;
}

// Profiler shows:
// BadList: 100ms
// ├── ListItem (id=1): 1ms
// ├── ListItem (id=2): 1ms
// ... 100 items × 1ms = 100ms total

// ✅ GOOD - memoized items
const MemoizedListItem = React.memo(ListItem);

function GoodList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(item => (
        <MemoizedListItem key={item.id} item={item} />
      ))}
    </ul>
  );
}

// Profiler now shows:
// GoodList: 5ms (only changed items render)
```

---

## Identifying Unnecessary Renders

Use DevTools "Highlight updates":

```typescript
// Component re-renders even when props don't change
function Child({ count }: { count: number }) {
  console.log('Child rendered');
  return <div>Count: {count}</div>;
}

function Parent() {
  const [text, setText] = useState('');
  const [count, setCount] = useState(0);

  return (
    <div>
      {/* Child re-renders when text changes! */}
      <input value={text} onChange={e => setText(e.target.value)} />
      <Child count={count} />
    </div>
  );
}

// Fix with React.memo
const MemoizedChild = React.memo(Child);
```

---

## Performance Budget

Set performance targets:

```typescript
const PERFORMANCE_BUDGET = {
  initialLoad: 1000,      // 1 second
  interaction: 100,       // 100ms
  animation: 16,          // 60fps
};

function checkPerformance(
  component: string,
  duration: number,
  phase: 'mount' | 'update'
) {
  const budget = phase === 'mount'
    ? PERFORMANCE_BUDGET.initialLoad
    : PERFORMANCE_BUDGET.interaction;

  if (duration > budget) {
    console.warn(
      `⚠️ ${component} exceeded budget: ${duration}ms > ${budget}ms`
    );
  }
}

// Use in Profiler
<Profiler
  id="App"
  onRender={(id, phase, actualDuration) => {
    checkPerformance(id, actualDuration, phase);
  }}
>
  <App />
</Profiler>
```

---

## Measuring User Interactions

```typescript
function Button() {
  const handleClick = () => {
    const startTime = performance.now();

    // Handle click
    updateState();

    const duration = performance.now() - startTime;

    // Report slow interactions
    if (duration > 100) {
      console.warn(`Slow click handler: ${duration}ms`);
    }
  };

  return <button onClick={handleClick}>Click</button>;
}
```

---

## Web Vitals Integration

```typescript
import { getCLS, getFID, getFCP, getLCP, getTTFB } from "web-vitals";

function reportWebVitals() {
  getCLS(console.log); // Cumulative Layout Shift
  getFID(console.log); // First Input Delay
  getFCP(console.log); // First Contentful Paint
  getLCP(console.log); // Largest Contentful Paint
  getTTFB(console.log); // Time to First Byte
}

// In App.tsx
useEffect(() => {
  reportWebVitals();
}, []);
```

---

## React DevTools Settings

**Profiler Settings:**

- ☑️ Record why each component rendered
- ☑️ Hide commits below (ms): 1
- ☑️ Highlight updates when components render

**General Settings:**

- ☑️ Highlight updates
- ☑️ Show inline warnings and errors

---

## Common Performance Issues

### **1. Expensive Render**

```typescript
// Problem: Expensive computation on every render
function Component({ data }) {
  const sorted = data.sort(); // Runs every render!
  return <List items={sorted} />;
}

// Solution: Memoize
const sorted = useMemo(() => data.sort(), [data]);
```

### **2. Large Component Tree**

```typescript
// Problem: Deep nesting causes cascading renders
<A><B><C><D><E /></D></C></B></A>

// Solution: Code split or flatten
const E = lazy(() => import('./E'));
```

### **3. Prop Drilling**

```typescript
// Problem: Props passed through many layers
<A config={config}>
  <B config={config}>
    <C config={config}>
      <D config={config} />

// Solution: Context or state management
const ConfigContext = createContext(config);
```

---

## Best Practices

✅ **Profile before optimizing** - measure first  
✅ **Focus on slow components** - optimize biggest gains  
✅ **Set performance budgets** - define targets  
✅ **Use React.memo** for expensive components  
✅ **Memoize expensive computations**  
✅ **Track Web Vitals** for real user metrics  
❌ **Don't optimize everything** - focus on bottlenecks  
❌ **Don't guess** - always measure

---

## Profiling Checklist

1. ✅ Open React DevTools Profiler
2. ✅ Record user interaction
3. ✅ Identify slow components (red/orange)
4. ✅ Check why they rendered
5. ✅ Apply memoization if needed
6. ✅ Re-profile to verify improvement
7. ✅ Repeat for next bottleneck

---

## Key Takeaways

1. **React DevTools Profiler** visualizes performance
2. **Flamegraph shows** component render hierarchy
3. **Ranked chart** prioritizes optimization work
4. **Profiler API** enables programmatic tracking
5. **Performance marks** measure custom timings
6. **Web Vitals** track real user experience
7. **Profile before optimizing** - don't guess!
