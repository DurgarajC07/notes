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

### **Stable Keys**

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
