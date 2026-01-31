# Reconciliation Algorithm in React

## Core Concept

The reconciliation algorithm (also called the "diffing algorithm") is how React determines the minimal set of changes needed to update the real DOM when the virtual DOM changes.

---

## The Diffing Heuristics

React uses a **heuristic O(n) algorithm** instead of the optimal O(n³) algorithm by making two assumptions:

1. **Different types of elements** produce different trees
2. **Keys** can hint which child elements may be stable across renders

---

## Different Element Types

When root elements have different types, React tears down the old tree and builds the new one from scratch.

```jsx
// Before
<div>
  <Counter />
</div>

// After
<span>
  <Counter />
</span>

// Result: <Counter /> is unmounted and remounted (state is lost!)
```

---

## Same Element Type

When elements have the same type, React keeps the same DOM node and only updates changed attributes.

```jsx
// Before
<div className="old" title="stuff" />

// After
<div className="new" title="stuff" />

// Result: Only className attribute is updated
```

---

## Component Elements

When updating component elements of the same type, the instance stays the same and state is maintained.

```jsx
// Before
<Counter count={1} />

// After
<Counter count={2} />

// Result: Same Counter instance, componentDidUpdate called with new props
```

---

## Recursing on Children

React iterates on children and generates a mutation when there's a difference.

### **Without Keys (Inefficient)**

```jsx
// Before
<ul>
  <li>A</li>
  <li>B</li>
</ul>

// After
<ul>
  <li>A</li>
  <li>B</li>
  <li>C</li>
</ul>

// React matches first two, inserts third - efficient
```

But inserting at the beginning is slow:

```jsx
// Before
<ul>
  <li>B</li>
  <li>C</li>
</ul>

// After
<ul>
  <li>A</li>
  <li>B</li>
  <li>C</li>
</ul>

// Without keys: React mutates all three children (inefficient!)
```

---

## Keys

Keys tell React which items correspond across renders, enabling efficient reordering.

```jsx
// Before
<ul>
  <li key="b">B</li>
  <li key="c">C</li>
</ul>

// After
<ul>
  <li key="a">A</li>
  <li key="b">B</li>
  <li key="c">C</li>
</ul>

// With keys: React knows 'b' and 'c' are the same, only inserts 'a' (efficient!)
```

---

## Key Anti-Patterns

### **❌ Using Index as Key**

```jsx
// BAD
items.map((item, index) => <Item key={index} {...item} />);

// Problem: Keys change when items reorder
// Before: [key=0: 'A', key=1: 'B', key=2: 'C']
// After:  [key=0: 'C', key=1: 'A', key=2: 'B']
// All items are "changed" even though content just moved!
```

### **✅ Using Stable IDs**

```jsx
// GOOD
items.map((item) => <Item key={item.id} {...item} />);
```

---

## Reconciliation in Action

```jsx
function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: "Learn React", done: false },
    { id: 2, text: "Build app", done: false },
  ]);

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input type="checkbox" checked={todo.done} />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}

// When todo.done changes from false to true:
// React: "li with key=1 still exists, just update checkbox attribute"
// Result: Minimal DOM mutation
```

---

## Performance Implications

### **Component Identity**

```jsx
// BAD - new component instance every render
function Parent() {
  return <Child />;  // New Child each time!
}

// GOOD - stable component identity
const Child = React.memo(function Child() { ... });
```

### **Component Identity Issues**

```typescript
// BAD - new component instance every render
function Parent() {
  const Child = () => <div>Child</div>; // ❌ New component every render!
  return <Child />;
}

// GOOD - stable component identity
const Child = () => <div>Child</div>;

function Parent() {
  return <Child />; // ✅ Same component instance
}
```

### **Inline Functions as Props**

```typescript
// BAD - breaks memoization
function Parent() {
  return (
    <ExpensiveChild
      onClick={() => console.log('clicked')} // ❌ New function every render
    />
  );
}

// GOOD - stable callback
function Parent() {
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);

  return <ExpensiveChild onClick={handleClick} />; // ✅ Same function
}
```

---

## Reconciliation Step-by-Step

### **Example: Todo List Update**

```typescript
// Initial render
function TodoList() {
  const todos = [
    { id: 1, text: 'Learn React' },
    { id: 2, text: 'Build app' }
  ];

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  );
}

// Virtual DOM (initial):
{
  type: 'ul',
  props: {},
  children: [
    { type: 'li', key: 1, children: ['Learn React'] },
    { type: 'li', key: 2, children: ['Build app'] }
  ]
}

// After adding todo at beginning:
const todos = [
  { id: 3, text: 'Setup project' },  // NEW
  { id: 1, text: 'Learn React' },
  { id: 2, text: 'Build app' }
];

// Virtual DOM (updated):
{
  type: 'ul',
  props: {},
  children: [
    { type: 'li', key: 3, children: ['Setup project'] }, // NEW
    { type: 'li', key: 1, children: ['Learn React'] },   // MOVED
    { type: 'li', key: 2, children: ['Build app'] }      // MOVED
  ]
}

// Reconciliation steps:
// 1. Compare 'ul' elements → Same type, keep DOM node
// 2. Compare children by keys:
//    - key=3: NEW → Create and insert DOM node
//    - key=1: EXISTS → Move existing DOM node
//    - key=2: EXISTS → Keep in place
// 3. Total DOM operations: 1 create, 1 insert (efficient!)
```

---

## Fiber Reconciliation Details

### **Work Loop**

```typescript
// Simplified Fiber reconciliation loop
function workLoop(deadline) {
  let shouldYield = false;

  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }

  if (nextUnitOfWork) {
    // More work to do, schedule next chunk
    requestIdleCallback(workLoop);
  } else if (pendingCommit) {
    // All work done, commit changes to DOM
    commitRoot();
  }
}

function performUnitOfWork(fiber) {
  // 1. Reconcile this fiber
  reconcileChildren(fiber);

  // 2. Return next unit of work (depth-first)
  if (fiber.child) return fiber.child;
  if (fiber.sibling) return fiber.sibling;
  return fiber.parent?.sibling;
}
```

### **Reconciliation Phases**

```typescript
// Phase 1: Render/Reconciliation (can be interrupted)
function reconcileChildren(fiber) {
  let oldFiber = fiber.alternate?.child;
  let prevSibling = null;

  fiber.props.children.forEach((element, index) => {
    let newFiber = null;

    // Check if same type
    const sameType = oldFiber && element.type === oldFiber.type;

    if (sameType) {
      // UPDATE - reuse DOM node
      newFiber = {
        type: element.type,
        props: element.props,
        dom: oldFiber.dom, // ✅ Reuse existing DOM
        parent: fiber,
        alternate: oldFiber,
        effectTag: "UPDATE",
      };
    } else {
      // PLACEMENT - create new DOM node
      if (element) {
        newFiber = {
          type: element.type,
          props: element.props,
          dom: null,
          parent: fiber,
          alternate: null,
          effectTag: "PLACEMENT",
        };
      }

      // DELETION - mark old node for removal
      if (oldFiber) {
        oldFiber.effectTag = "DELETION";
        deletions.push(oldFiber);
      }
    }

    // Link siblings
    if (index === 0) {
      fiber.child = newFiber;
    } else {
      prevSibling.sibling = newFiber;
    }

    prevSibling = newFiber;
    oldFiber = oldFiber?.sibling;
  });
}

// Phase 2: Commit (synchronous, cannot be interrupted)
function commitRoot() {
  deletions.forEach(commitWork);
  commitWork(wipRoot.child);
  currentRoot = wipRoot;
  wipRoot = null;
}

function commitWork(fiber) {
  if (!fiber) return;

  const domParent = fiber.parent.dom;

  if (fiber.effectTag === "PLACEMENT" && fiber.dom) {
    domParent.appendChild(fiber.dom);
  } else if (fiber.effectTag === "UPDATE" && fiber.dom) {
    updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  } else if (fiber.effectTag === "DELETION") {
    domParent.removeChild(fiber.dom);
  }

  commitWork(fiber.child);
  commitWork(fiber.sibling);
}
```

---

## Key Strategies

### **1. Tree Diffing**

```typescript
// React compares trees level by level
// Time complexity: O(n) instead of O(n³)

// Before:
<div>
  <span>Hello</span>
</div>

// After:
<div>
  <p>Hello</p>
</div>

// Algorithm:
// 1. Compare root: div === div ✅ Keep
// 2. Compare children: span !== p ❌ Replace
// Result: Destroy <span>, create <p>
```

### **2. Component Type Comparison**

```typescript
// Different component types = full remount
function App() {
  const [view, setView] = useState('list');

  return (
    <div>
      {view === 'list' ? (
        <ListView items={items} />  // ❌ Unmounts when switching
      ) : (
        <GridView items={items} />  // ❌ Unmounts when switching
      )}
    </div>
  );
}

// State is lost because components are different types!

// Better: Use same component with prop
function App() {
  const [viewMode, setViewMode] = useState('list');

  return (
    <ItemView mode={viewMode} items={items} /> // ✅ Keeps state
  );
}
```

### **3. Keys for List Optimization**

```typescript
// Example: Reordering list items
const [items, setItems] = useState([
  { id: 'a', name: 'Alice' },
  { id: 'b', name: 'Bob' },
  { id: 'c', name: 'Charlie' }
]);

// Without keys (index-based):
items.map((item, index) => (
  <UserCard key={index} user={item} />
));

// When reordering to ['b', 'c', 'a']:
// React thinks:
// - index 0: Alice → Bob (UPDATE with new props)
// - index 1: Bob → Charlie (UPDATE with new props)
// - index 2: Charlie → Alice (UPDATE with new props)
// Result: 3 updates, state might be confused

// With keys (id-based):
items.map(item => (
  <UserCard key={item.id} user={item} />
));

// When reordering to ['b', 'c', 'a']:
// React knows:
// - id='b': MOVE to position 0
// - id='c': MOVE to position 1
// - id='a': MOVE to position 2
// Result: 0 updates, just DOM moves, state preserved
```

---

## Common Pitfalls

### **Pitfall 1: Unstable Keys**

```typescript
// ❌ BAD: Key changes on every render
<div key={Math.random()}>Content</div>
<div key={Date.now()}>Content</div>

// React sees different key → unmounts and remounts
// State lost, expensive operations repeated
```

### **Pitfall 2: Non-Unique Keys**

```typescript
// ❌ BAD: Duplicate keys
items.map(item => (
  <div key="same-key">{item.name}</div>
));

// React warning: "Keys should be unique among siblings"
// Reconciliation breaks, unpredictable behavior
```

### **Pitfall 3: Index as Key with Dynamic Lists**

```typescript
// ❌ BAD: Index keys with reordering/filtering
const [items, setItems] = useState(['A', 'B', 'C']);

items.map((item, index) => (
  <input key={index} defaultValue={item} />
));

// User types in inputs: ['A-typed', 'B-typed', 'C-typed']
// Then delete 'A':
// Before: [0:'A-typed', 1:'B-typed', 2:'C-typed']
// After:  [0:'B', 1:'C']
// React thinks:
// - index 0: 'A' → 'B' (reuses input with 'A-typed' value!)
// - index 1: 'B' → 'C' (reuses input with 'B-typed' value!)
// - index 2: removed
// Result: Wrong values in wrong inputs!

// ✅ GOOD: Stable unique keys
items.map(item => (
  <input key={item.id} defaultValue={item.name} />
));
```

### **Pitfall 4: Changing Component Types**

```typescript
// ❌ BAD: Conditional component types
function Display({ isSpecial, data }) {
  return isSpecial ? (
    <SpecialView data={data} />
  ) : (
    <NormalView data={data} />
  );
}

// Toggling isSpecial unmounts one, mounts the other
// State lost, useEffect cleanup runs

// ✅ GOOD: Use props to control behavior
function Display({ isSpecial, data }) {
  return <UnifiedView special={isSpecial} data={data} />;
}
```

---

## Optimization Techniques

### **React.memo for Component Memoization**

```typescript
// Expensive component that shouldn't re-render unnecessarily
const ExpensiveComponent = React.memo(({ data }) => {
  console.log('Rendering ExpensiveComponent');
  return <div>{expensiveCalculation(data)}</div>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const data = { value: 42 }; // ⚠️ New object every render

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      <ExpensiveComponent data={data} />
      {/* Still re-renders because data is new object! */}
    </>
  );
}

// ✅ Fixed with useMemo
function Parent() {
  const [count, setCount] = useState(0);
  const data = useMemo(() => ({ value: 42 }), []);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <ExpensiveComponent data={data} />
      {/* Now skips re-render when count changes */}
    </>
  );
}
```

### **Key Strategies for Lists**

```typescript
// Strategy 1: Server-provided IDs (best)
items.map(item => <Item key={item.id} {...item} />);

// Strategy 2: Client-generated stable IDs
const itemsWithIds = items.map((item, i) => ({
  ...item,
  _id: item.id || `generated-${i}`
}));

// Strategy 3: Hash of content (for immutable data)
items.map(item => <Item key={hash(item)} {...item} />);

// Strategy 4: Index (ONLY for static lists)
// Only OK if list never reorders, filters, or adds items
staticItems.map((item, index) => <Item key={index} {...item} />);
```

### **Fragment Keys**

```typescript
// When returning multiple elements, use Fragment with key
function Glossary({ items }) {
  return (
    <dl>
      {items.map(item => (
        <React.Fragment key={item.id}>
          <dt>{item.term}</dt>
          <dd>{item.description}</dd>
        </React.Fragment>
      ))}
    </dl>
  );
}
```

---

## Debugging Reconciliation

### **React DevTools Profiler**

```typescript
// Wrap components to measure reconciliation time
function App() {
  return (
    <Profiler id="TodoList" onRender={onRenderCallback}>
      <TodoList />
    </Profiler>
  );
}

function onRenderCallback(
  id, // "TodoList"
  phase, // "mount" or "update"
  actualDuration, // Time spent rendering
  baseDuration, // Estimated time without memoization
  startTime,
  commitTime
) {
  console.log(`${id} ${phase} took ${actualDuration}ms`);
}
```

### **Highlight Updates**

```typescript
// In React DevTools:
// Settings → General → Highlight updates when components render
// Shows colored borders when components reconcile
```

### **Why Did You Render**

```typescript
// Library to track unnecessary re-renders
import whyDidYouRender from "@welldone-software/why-did-you-render";

if (process.env.NODE_ENV === "development") {
  whyDidYouRender(React, {
    trackAllPureComponents: true,
  });
}

// Mark specific components to track
MyComponent.whyDidYouRender = true;
```

---

## Real-World Example: Chat Application

```typescript
interface Message {
  id: string;
  text: string;
  userId: string;
  timestamp: number;
}

function ChatMessages({ messages }: { messages: Message[] }) {
  return (
    <div className="messages">
      {messages.map(message => (
        // ✅ Use message.id as key
        <MessageBubble
          key={message.id}
          message={message}
        />
      ))}
    </div>
  );
}

// When new message arrives:
// 1. React compares old and new message lists
// 2. Sees new key in list
// 3. Creates DOM node for new message only
// 4. Keeps all existing message nodes unchanged
// Result: Optimal performance, smooth scrolling

const MessageBubble = React.memo(({ message }) => {
  // Memoized so unchanged messages don't re-render
  return (
    <div className="message">
      <span className="user">{message.userId}</span>
      <p>{message.text}</p>
      <time>{new Date(message.timestamp).toLocaleString()}</time>
    </div>
  );
});
```

---

```jsx
// BAD - random keys break reconciliation
{
  items.map((item) => <Item key={Math.random()} />);
}

// GOOD - stable, unique keys
{
  items.map((item) => <Item key={item.id} />);
}
```

---

## Best Practices

1. **Always use stable, unique keys for lists**
   - Prefer server IDs over generated IDs
   - Never use array index for dynamic lists
   - Never use Math.random() or Date.now()

2. **Keep component types stable**
   - Avoid conditional component types
   - Use props to control behavior, not different components

3. **Memoize expensive components**
   - Use React.memo for components with expensive renders
   - Combine with useMemo/useCallback for props

4. **Minimize prop changes**
   - Avoid creating new objects/arrays/functions inline
   - Use useMemo/useCallback for referential stability

5. **Profile reconciliation performance**
   - Use React DevTools Profiler
   - Measure actualDuration in Profiler onRender
   - Enable "Highlight updates" to see unnecessary renders

6. **Understand reconciliation phases**
   - Render phase is interruptible (use for expensive work)
   - Commit phase is synchronous (keep it fast)

7. **Use Fragments with keys when needed**
   - When mapping to multiple elements
   - Avoids unnecessary wrapper divs

8. **Avoid breaking component identity**
   - Don't define components inside other components
   - Don't create new component types on every render

9. **Test with React Strict Mode**
   - Helps catch issues with component identity
   - Double-invokes render to expose side effects

10. **Monitor bundle size**
    - Large component trees = more reconciliation work
    - Code split to reduce initial tree size

---

## Key Takeaways

1. **React's reconciliation is an O(n) diffing algorithm** that compares virtual DOM trees to minimize actual DOM updates

2. **Keys are critical for list reconciliation** - they help React identify which items have changed, been added, or removed

3. **Component identity matters** - same component type preserves state, different type triggers unmount/mount

4. **Fiber architecture enables concurrent rendering** - reconciliation can be paused and resumed

5. **Two-phase reconciliation**: interruptible render phase (builds work-in-progress tree) + synchronous commit phase (applies DOM updates)

6. **Index keys are dangerous for dynamic lists** - they can cause state to be associated with wrong elements

7. **React.memo prevents unnecessary reconciliation** - but only if props remain referentially equal

8. **Reconciliation can be the performance bottleneck** - profile with DevTools to identify issues

9. **Understanding reconciliation helps debug weird behavior** - like state appearing in wrong components or unexpected unmounts

10. **Best practice: stable keys, stable types, memoized props** - these three principles cover 90% of reconciliation optimization

---

✅ **Always use stable, unique keys** for lists  
✅ **Avoid changing element types** (div → span)  
✅ **Use React.memo** for expensive components  
✅ **Keep component tree structure stable**  
❌ **Never use index as key** for dynamic lists  
❌ **Don't use random values as keys**  
❌ **Don't change key values** unnecessarily

---

## Key Takeaways

1. **React compares trees level-by-level** (breadth-first)
2. **Different types → full remount** (state lost)
3. **Same type → update props** (state preserved)
4. **Keys enable efficient reordering** of lists
5. **Stable keys are critical** for performance
6. **Index keys break** when list order changes
