# React.memo and Memoization

## Core Concept

Memoization prevents unnecessary re-renders by caching results and only recomputing when dependencies change. React provides three memoization tools: `React.memo`, `useMemo`, and `useCallback`.

---

## React.memo

Memoizes entire component - only re-renders if props change:

```typescript
interface Props {
  name: string;
  age: number;
}

// Without memo - re-renders on every parent render
function ExpensiveComponent({ name, age }: Props) {
  console.log('Rendering ExpensiveComponent');
  return <div>{name} is {age} years old</div>;
}

// With memo - only re-renders when name or age change
const MemoizedComponent = React.memo(ExpensiveComponent);

// Custom comparison
const MemoizedWithCompare = React.memo(
  ExpensiveComponent,
  (prevProps, nextProps) => {
    // Return true if props are equal (skip render)
    return prevProps.name === nextProps.name &&
           prevProps.age === nextProps.age;
  }
);
```

---

## useMemo

Memoizes expensive computations:

```typescript
function ProductList({ products, filter }: Props) {
  // Without useMemo - filters on every render
  const filteredProducts = products.filter(p => p.category === filter);

  // With useMemo - only filters when dependencies change
  const filteredProducts = useMemo(
    () => products.filter(p => p.category === filter),
    [products, filter]
  );

  return (
    <ul>
      {filteredProducts.map(p => <li key={p.id}>{p.name}</li>)}
    </ul>
  );
}
```

---

## useCallback

Memoizes function references:

```typescript
function Parent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // Without useCallback - new function every render
  const handleClick = () => {
    console.log('Clicked!');
  };

  // With useCallback - same function reference
  const handleClickMemo = useCallback(() => {
    console.log('Clicked!');
  }, []); // No dependencies - never changes

  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>

      {/* Re-renders even when text changes! */}
      <Child onClick={handleClick} />

      {/* Only re-renders when handleClickMemo changes */}
      <MemoizedChild onClick={handleClickMemo} />
    </>
  );
}

const MemoizedChild = React.memo(Child);
```

---

## When to Use Memoization

### **✅ Use When:**

1. **Expensive computations** that slow down renders
2. **Child components that re-render unnecessarily**
3. **Passing callbacks to memoized children**
4. **Large lists** or data transformations

### **❌ Don't Use When:**

1. **Cheap computations** - overhead > benefit
2. **Props always change** - memo never helps
3. **Over-optimization** - premature optimization is bad
4. **Small components** - not worth complexity

---

## Real-World Example: Data Table

```typescript
interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

interface Props {
  products: Product[];
  sortBy: 'name' | 'price';
  filterCategory?: string;
}

function ProductTable({ products, sortBy, filterCategory }: Props) {
  // Memoize filtering
  const filtered = useMemo(() => {
    return filterCategory
      ? products.filter(p => p.category === filterCategory)
      : products;
  }, [products, filterCategory]);

  // Memoize sorting
  const sorted = useMemo(() => {
    return [...filtered].sort((a, b) => {
      return sortBy === 'name'
        ? a.name.localeCompare(b.name)
        : a.price - b.price;
    });
  }, [filtered, sortBy]);

  // Memoize row renderer
  const renderRow = useCallback((product: Product) => (
    <ProductRow key={product.id} product={product} />
  ), []);

  return (
    <table>
      <tbody>
        {sorted.map(renderRow)}
      </tbody>
    </table>
  );
}

// Memoize row component
const ProductRow = React.memo(({ product }: { product: Product }) => (
  <tr>
    <td>{product.name}</td>
    <td>${product.price}</td>
  </tr>
));
```

---

## Common Pitfalls

### **1. Inline Objects/Arrays**

```typescript
// ❌ BAD - creates new object every render
<MemoizedChild config={{ theme: 'dark' }} />

// ✅ GOOD - memoize config
const config = useMemo(() => ({ theme: 'dark' }), []);
<MemoizedChild config={config} />

// Or define outside component
const CONFIG = { theme: 'dark' };
<MemoizedChild config={CONFIG} />
```

### **2. Unnecessary Dependencies**

```typescript
// ❌ BAD - changes on every render
const value = useMemo(() => expensiveComputation(data), [data, Math.random()]);

// ✅ GOOD - only real dependencies
const value = useMemo(() => expensiveComputation(data), [data]);
```

### **3. Forgetting Dependencies**

```typescript
// ❌ BAD - stale closure
const handleClick = useCallback(() => {
  console.log(count); // Always logs initial count!
}, []);

// ✅ GOOD - include dependencies
const handleClick = useCallback(() => {
  console.log(count);
}, [count]);
```

---

## Measuring Impact

```typescript
import { Profiler } from 'react';

function App() {
  const onRender = (
    id: string,
    phase: "mount" | "update",
    actualDuration: number
  ) => {
    console.log(`${id} (${phase}): ${actualDuration}ms`);
  };

  return (
    <Profiler id="ProductList" onRender={onRender}>
      <ProductList />
    </Profiler>
  );
}
```

---

## React DevTools Profiler

Use React DevTools to identify:

- Components that render frequently
- Components with long render times
- Unnecessary re-renders

Look for:

- 🟥 Red/orange = slow renders
- 🟩 Green = fast renders
- Gray = did not render

---

## Best Practices

✅ **Profile before optimizing** - measure actual impact  
✅ **Memoize expensive computations** with useMemo  
✅ **Memoize callbacks** passed to memoized children  
✅ **Use React.memo** for pure components  
✅ **Extract static objects** outside components  
❌ **Don't memoize everything** - overhead exists  
❌ **Don't optimize prematurely** - profile first  
❌ **Don't forget dependencies** - causes bugs

---

## Key Takeaways

1. **React.memo** prevents component re-renders
2. **useMemo** caches expensive computation results
3. **useCallback** caches function references
4. **Only optimize when needed** - profile first
5. **Dependencies matter** - wrong deps cause bugs
6. **Inline objects break memoization**
7. **Memoization has overhead** - use judiciously
