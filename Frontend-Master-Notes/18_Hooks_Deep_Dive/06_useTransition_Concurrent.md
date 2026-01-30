# useTransition and Concurrent Features

## Core Concept

React 18 introduced concurrent features enabling non-blocking UI updates. `useTransition` marks updates as low-priority, allowing high-priority updates (like typing) to interrupt them. `useDeferredValue` creates a lagging version of a value for expensive computations.

---

## useTransition Basics

```typescript
import { useTransition, useState } from 'react';

function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (value: string) => {
    setQuery(value); // Immediate update

    // Mark as low-priority transition
    startTransition(() => {
      const filtered = searchData(value); // Expensive operation
      setResults(filtered);
    });
  };

  return (
    <div>
      <input
        value={query}
        onChange={(e) => handleSearch(e.target.value)}
        placeholder="Search..."
      />

      {isPending && <div>Searching...</div>}

      <ul>
        {results.map((result, i) => (
          <li key={i}>{result}</li>
        ))}
      </ul>
    </div>
  );
}
```

---

## Priority Levels

```typescript
// High priority (immediate)
function HighPriority() {
  const [count, setCount] = useState(0);

  // Immediate update - blocks UI
  const handleClick = () => {
    setCount(c => c + 1);
  };

  return <button onClick={handleClick}>{count}</button>;
}

// Low priority (transition)
function LowPriority() {
  const [items, setItems] = useState<number[]>([]);
  const [isPending, startTransition] = useTransition();

  // Non-blocking update - can be interrupted
  const handleAdd = () => {
    startTransition(() => {
      setItems(prev => [...prev, prev.length]);
    });
  };

  return (
    <div>
      <button onClick={handleAdd}>
        Add {isPending && '(pending...)'}
      </button>
      <div>{items.length} items</div>
    </div>
  );
}
```

---

## useDeferredValue

```typescript
import { useDeferredValue, useState, useMemo } from 'react';

function SearchWithDeferred() {
  const [query, setQuery] = useState('');

  // Create deferred (lagging) version
  const deferredQuery = useDeferredValue(query);

  // Use deferred value for expensive computation
  const results = useMemo(() => {
    return expensiveSearch(deferredQuery);
  }, [deferredQuery]);

  // Check if deferred value is catching up
  const isStale = query !== deferredQuery;

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
      />

      <div style={{ opacity: isStale ? 0.5 : 1 }}>
        {results.map(result => (
          <div key={result.id}>{result.name}</div>
        ))}
      </div>
    </div>
  );
}
```

---

## Comparison: useTransition vs useDeferredValue

```typescript
// useTransition - control when state updates
function WithTransition() {
  const [input, setInput] = useState('');
  const [list, setList] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (value: string) => {
    setInput(value); // Immediate

    startTransition(() => {
      setList(generateList(value)); // Deferred
    });
  };

  return (
    <>
      <input value={input} onChange={e => handleChange(e.target.value)} />
      {isPending && <Spinner />}
      <List items={list} />
    </>
  );
}

// useDeferredValue - defer a value
function WithDeferredValue() {
  const [input, setInput] = useState('');
  const deferredInput = useDeferredValue(input);

  // Generate list with deferred value
  const list = useMemo(() => {
    return generateList(deferredInput);
  }, [deferredInput]);

  return (
    <>
      <input value={input} onChange={e => setInput(e.target.value)} />
      <List items={list} />
    </>
  );
}
```

---

## Tab Switching

```typescript
interface Tab {
  id: string;
  label: string;
  content: ReactNode;
}

function Tabs({ tabs }: { tabs: Tab[] }) {
  const [activeTab, setActiveTab] = useState(tabs[0].id);
  const [isPending, startTransition] = useTransition();

  const handleTabClick = (tabId: string) => {
    startTransition(() => {
      setActiveTab(tabId); // Can be interrupted
    });
  };

  const currentTab = tabs.find(tab => tab.id === activeTab);

  return (
    <div>
      <div className="tabs">
        {tabs.map(tab => (
          <button
            key={tab.id}
            onClick={() => handleTabClick(tab.id)}
            disabled={isPending && tab.id !== activeTab}
          >
            {tab.label}
          </button>
        ))}
      </div>

      <div className="tab-content" style={{ opacity: isPending ? 0.5 : 1 }}>
        {currentTab?.content}
      </div>
    </div>
  );
}
```

---

## Suspense Integration

```typescript
import { Suspense, useTransition } from 'react';

// Suspending component
function SlowComponent({ query }: { query: string }) {
  const data = use(fetchData(query)); // Suspends while loading

  return <div>{data.map(item => <div key={item.id}>{item.name}</div>)}</div>;
}

function App() {
  const [query, setQuery] = useState('');
  const [deferredQuery, setDeferredQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  const handleSearch = (value: string) => {
    setQuery(value); // Immediate update to input

    startTransition(() => {
      setDeferredQuery(value); // Transition for Suspense
    });
  };

  return (
    <div>
      <input
        value={query}
        onChange={e => handleSearch(e.target.value)}
      />

      <Suspense fallback={<div>Loading...</div>}>
        {isPending ? (
          <div>Searching...</div>
        ) : (
          <SlowComponent query={deferredQuery} />
        )}
      </Suspense>
    </div>
  );
}
```

---

## Throttling with Transition

```typescript
function ThrottledList() {
  const [items, setItems] = useState<number[]>([]);
  const [isPending, startTransition] = useTransition();

  const addMany = () => {
    // Add items in transition to prevent blocking
    startTransition(() => {
      setItems(prev => {
        const newItems = [...prev];
        for (let i = 0; i < 10000; i++) {
          newItems.push(i);
        }
        return newItems;
      });
    });
  };

  return (
    <div>
      <button onClick={addMany}>
        Add 10,000 items {isPending && '(adding...)'}
      </button>

      <div>
        {items.map((item, i) => (
          <div key={i}>{item}</div>
        ))}
      </div>
    </div>
  );
}
```

---

## Real-World: Product Filter

```typescript
interface Product {
  id: string;
  name: string;
  category: string;
  price: number;
}

function ProductList({ products }: { products: Product[] }) {
  const [search, setSearch] = useState('');
  const [category, setCategory] = useState('all');
  const [isPending, startTransition] = useTransition();

  // Deferred values for expensive filtering
  const deferredSearch = useDeferredValue(search);
  const deferredCategory = useDeferredValue(category);

  const filteredProducts = useMemo(() => {
    let filtered = products;

    if (deferredCategory !== 'all') {
      filtered = filtered.filter(p => p.category === deferredCategory);
    }

    if (deferredSearch) {
      filtered = filtered.filter(p =>
        p.name.toLowerCase().includes(deferredSearch.toLowerCase())
      );
    }

    return filtered;
  }, [products, deferredSearch, deferredCategory]);

  const isStale = search !== deferredSearch || category !== deferredCategory;

  return (
    <div>
      <input
        value={search}
        onChange={e => setSearch(e.target.value)}
        placeholder="Search products..."
      />

      <select
        value={category}
        onChange={e => {
          startTransition(() => {
            setCategory(e.target.value);
          });
        }}
      >
        <option value="all">All Categories</option>
        <option value="electronics">Electronics</option>
        <option value="clothing">Clothing</option>
      </select>

      <div style={{ opacity: isStale ? 0.6 : 1 }}>
        {filteredProducts.length === 0 ? (
          <div>No products found</div>
        ) : (
          <ul>
            {filteredProducts.map(product => (
              <li key={product.id}>
                {product.name} - ${product.price}
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
}
```

---

## Real-World: Autocomplete

```typescript
function Autocomplete() {
  const [input, setInput] = useState('');
  const [suggestions, setSuggestions] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleInput = (value: string) => {
    setInput(value); // Immediate - keep input responsive

    startTransition(() => {
      // Expensive search through large dataset
      const results = searchDatabase(value);
      setSuggestions(results);
    });
  };

  return (
    <div className="autocomplete">
      <input
        value={input}
        onChange={e => handleInput(e.target.value)}
        placeholder="Type to search..."
      />

      {isPending && <div className="loading">Searching...</div>}

      {suggestions.length > 0 && (
        <ul className="suggestions">
          {suggestions.map((suggestion, i) => (
            <li key={i} onClick={() => setInput(suggestion)}>
              {suggestion}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## Performance Comparison

```typescript
// ❌ Without transition - blocks UI
function WithoutTransition() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);

  const handleSearch = (value: string) => {
    setQuery(value);
    setResults(expensiveSearch(value)); // Blocks typing
  };

  return (
    <input value={query} onChange={e => handleSearch(e.target.value)} />
  );
}

// ✅ With transition - stays responsive
function WithTransition() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (value: string) => {
    setQuery(value); // Immediate

    startTransition(() => {
      setResults(expensiveSearch(value)); // Non-blocking
    });
  };

  return (
    <input value={query} onChange={e => handleSearch(e.target.value)} />
  );
}
```

---

## Best Practices

✅ **Use for expensive state updates**  
✅ **Keep input responsive** - don't wrap in transition  
✅ **Show pending state** to users  
✅ **Combine with Suspense** for data fetching  
✅ **Use `useDeferredValue`** for derived values  
✅ **Apply to navigation** and tab switching  
❌ **Don't wrap every state update**  
❌ **Don't use for critical updates**  
❌ **Don't forget pending indicators**

---

## Key Takeaways

1. **useTransition marks updates as low-priority**
2. **useDeferredValue creates lagging value**
3. **Keeps UI responsive** during expensive updates
4. **Works with Suspense** for async data
5. **Great for search, filtering, navigation**
6. **Can be interrupted** by high-priority updates
7. **React 18+ only**
