# ⚡ useEffect & Component Lifecycle

> useEffect is React's side-effect hook. Understanding its timing, cleanup, and dependencies is critical for preventing bugs and memory leaks in production applications.

---

## 📖 1. Concept Explanation

### What is useEffect?

**useEffect** synchronizes a component with external systems (APIs, subscriptions, DOM, timers).

```jsx
useEffect(() => {
  // Side effect runs after render
  document.title = `Count: ${count}`;

  // Optional cleanup
  return () => {
    document.title = "App";
  };
}, [count]); // Dependencies
```

### Effect Timing

```
Render → Commit to DOM → Browser Paint → useEffect runs
```

**vs useLayoutEffect:**

```
Render → Commit to DOM → useLayoutEffect runs → Browser Paint
```

---

## 🧠 2. Why It Matters

### Real-World Impact

**Memory leaks:**

```jsx
// ❌ LEAK: Event listener not cleaned up
useEffect(() => {
  window.addEventListener("resize", handleResize);
  // Missing cleanup!
}, []);

// Component unmounts → listener still active → memory leak
```

**Stale closures:**

```jsx
// ❌ BUG: Stale count value
useEffect(() => {
  const interval = setInterval(() => {
    console.log(count); // Always logs initial count!
  }, 1000);

  return () => clearInterval(interval);
}, []); // Empty deps → closure captures initial count
```

**Infinite loops:**

```jsx
// ❌ INFINITE LOOP
useEffect(() => {
  setCount(count + 1); // Re-renders → runs effect → re-renders...
}, [count]);
```

---

## ⚙️ 3. Effect Execution Order

### Mount Phase

```jsx
function Component() {
  console.log("1. Render");

  useEffect(() => {
    console.log("3. Effect runs");

    return () => {
      console.log("(Cleanup will run on unmount/re-run)");
    };
  });

  console.log("2. Render complete");
}

// Output on mount:
// 1. Render
// 2. Render complete
// 3. Effect runs
```

### Update Phase

```jsx
function Component({ prop }) {
  console.log("1. Render with new prop");

  useEffect(() => {
    console.log("4. Effect runs with new prop");

    return () => {
      console.log("3. Cleanup runs (previous effect)");
    };
  }, [prop]);

  console.log("2. Render complete");
}

// Output on update:
// 1. Render with new prop
// 2. Render complete
// 3. Cleanup runs (previous effect)
// 4. Effect runs with new prop
```

### Unmount Phase

```jsx
function Component() {
  useEffect(() => {
    return () => {
      console.log("Cleanup on unmount");
    };
  }, []);
}

// Component unmounts → cleanup runs
```

---

## ✅ 4. Best Practices

### DO ✅

**1. Always clean up side effects:**

```jsx
// Event listeners
useEffect(() => {
  function handleScroll() {
    console.log(window.scrollY);
  }

  window.addEventListener("scroll", handleScroll);

  return () => {
    window.removeEventListener("scroll", handleScroll);
  };
}, []);

// Timers
useEffect(() => {
  const timer = setTimeout(() => {
    console.log("Delayed action");
  }, 1000);

  return () => clearTimeout(timer);
}, []);

// Subscriptions
useEffect(() => {
  const subscription = api.subscribe(handleData);

  return () => {
    subscription.unsubscribe();
  };
}, []);

// Async requests
useEffect(() => {
  const controller = new AbortController();

  fetch("/api/data", { signal: controller.signal })
    .then((res) => res.json())
    .then(setData)
    .catch((error) => {
      if (error.name !== "AbortError") {
        console.error(error);
      }
    });

  return () => controller.abort();
}, []);
```

**2. Declare all dependencies:**

```jsx
// ✅ CORRECT: count and multiplier in deps
useEffect(() => {
  console.log(count * multiplier);
}, [count, multiplier]);

// ❌ WRONG: Missing multiplier
useEffect(() => {
  console.log(count * multiplier); // Stale multiplier!
}, [count]);
```

**3. Use functional updates to avoid dependencies:**

```jsx
// ❌ Needs count in deps
useEffect(() => {
  const interval = setInterval(() => {
    setCount(count + 1);
  }, 1000);

  return () => clearInterval(interval);
}, [count]); // Effect runs every second!

// ✅ No dependency needed
useEffect(() => {
  const interval = setInterval(() => {
    setCount((c) => c + 1); // Functional update
  }, 1000);

  return () => clearInterval(interval);
}, []); // Effect runs once
```

**4. Separate concerns into multiple effects:**

```jsx
// ❌ Mixed concerns
useEffect(() => {
  // Subscription
  const sub = api.subscribe(handleData);

  // Event listener
  window.addEventListener("resize", handleResize);

  // Timer
  const timer = setInterval(updateTime, 1000);

  return () => {
    sub.unsubscribe();
    window.removeEventListener("resize", handleResize);
    clearInterval(timer);
  };
}, []);

// ✅ Separate effects
useEffect(() => {
  const sub = api.subscribe(handleData);
  return () => sub.unsubscribe();
}, []);

useEffect(() => {
  window.addEventListener("resize", handleResize);
  return () => window.removeEventListener("resize", handleResize);
}, []);

useEffect(() => {
  const timer = setInterval(updateTime, 1000);
  return () => clearInterval(timer);
}, []);
```

---

### DON'T ❌

```jsx
// ❌ Don't mutate state directly
useEffect(() => {
  data.count++; // Mutation!
}, [data]);

// ✅ Use setState
useEffect(() => {
  setData((prev) => ({ ...prev, count: prev.count + 1 }));
}, []);

// ❌ Don't use async directly
useEffect(async () => {
  // ERROR!
  const data = await fetchData();
}, []);

// ✅ Create async function inside
useEffect(() => {
  async function fetchData() {
    const data = await fetch("/api/data");
    setData(data);
  }

  fetchData();
}, []);

// ❌ Don't ignore ESLint exhaustive-deps
useEffect(() => {
  console.log(value);
}, []); // eslint-disable-next-line react-hooks/exhaustive-deps

// ✅ Fix the dependency or use callback ref
const callbackRef = useRef(callback);
callbackRef.current = callback;

useEffect(() => {
  callbackRef.current();
}, []);
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Stale Closure

```jsx
// ❌ WRONG: Stale count
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      console.log(count); // Always logs 0!
      setCount(count + 1); // Always sets to 1!
    }, 1000);

    return () => clearInterval(interval);
  }, []); // Empty deps → closure captures initial count

  return <div>{count}</div>;
}

// ✅ CORRECT: Functional update
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setCount((c) => c + 1); // Always current count
    }, 1000);

    return () => clearInterval(interval);
  }, []);

  return <div>{count}</div>;
}
```

---

### Mistake #2: Missing Cleanup

```jsx
// ❌ MEMORY LEAK
function Component() {
  useEffect(() => {
    const ws = new WebSocket("wss://api.example.com");

    ws.onmessage = (event) => {
      console.log(event.data);
    };

    // No cleanup! WebSocket stays open after unmount
  }, []);
}

// ✅ CLEANED UP
function Component() {
  useEffect(() => {
    const ws = new WebSocket("wss://api.example.com");

    ws.onmessage = (event) => {
      console.log(event.data);
    };

    return () => {
      ws.close(); // Clean up WebSocket
    };
  }, []);
}
```

---

### Mistake #3: Race Condition

```jsx
// ❌ RACE CONDITION
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then(setUser);

    // Problem: userId changes to 2 while request for 1 is in flight
    // Request for 1 completes AFTER request for 2
    // Shows user 1's data!
  }, [userId]);
}

// ✅ FIX: Ignore stale responses
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let ignore = false;

    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then((data) => {
        if (!ignore) {
          setUser(data);
        }
      });

    return () => {
      ignore = true; // Ignore stale response
    };
  }, [userId]);
}

// ✅ BETTER: Use AbortController
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then((res) => res.json())
      .then(setUser)
      .catch((error) => {
        if (error.name !== "AbortError") {
          console.error(error);
        }
      });

    return () => {
      controller.abort(); // Cancel in-flight request
    };
  }, [userId]);
}
```

---

### Mistake #4: Infinite Loop

```jsx
// ❌ INFINITE LOOP: Object dependency
function Component() {
  const [data, setData] = useState({ count: 0 });

  useEffect(() => {
    setData({ count: data.count + 1 }); // New object every time!
  }, [data]); // data changes → effect runs → data changes...
}

// ✅ FIX: Only depend on specific property
function Component() {
  const [data, setData] = useState({ count: 0 });

  useEffect(() => {
    setData((prev) => ({ ...prev, count: prev.count + 1 }));
  }, []); // Run once
}

// ❌ INFINITE LOOP: Function dependency
function Component() {
  const handleClick = () => {
    console.log("Clicked");
  };

  useEffect(() => {
    document.addEventListener("click", handleClick);
    return () => document.removeEventListener("click", handleClick);
  }, [handleClick]); // New function every render!
}

// ✅ FIX: useCallback
function Component() {
  const handleClick = useCallback(() => {
    console.log("Clicked");
  }, []);

  useEffect(() => {
    document.addEventListener("click", handleClick);
    return () => document.removeEventListener("click", handleClick);
  }, [handleClick]); // Stable reference
}
```

---

## 🧪 6. Advanced Patterns

### Pattern 1: Custom Hook for Data Fetching

```typescript
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    async function fetchData() {
      try {
        setLoading(true);
        const res = await fetch(url, { signal: controller.signal });
        const json = await res.json();
        setData(json);
        setError(null);
      } catch (err) {
        if (err.name !== 'AbortError') {
          setError(err as Error);
        }
      } finally {
        setLoading(false);
      }
    }

    fetchData();

    return () => controller.abort();
  }, [url]);

  return { data, loading, error };
}

// Usage
function UserProfile({ userId }: { userId: string }) {
  const { data, loading, error } = useFetch<User>(`/api/users/${userId}`);

  if (loading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  return <div>{data?.name}</div>;
}
```

---

### Pattern 2: Debounced Effect

```typescript
function useDebouncedEffect(
  effect: () => void | (() => void),
  deps: any[],
  delay: number
) {
  useEffect(() => {
    const timer = setTimeout(() => {
      effect();
    }, delay);

    return () => clearTimeout(timer);
  }, [...deps, delay]);
}

// Usage
function SearchBox() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  useDebouncedEffect(() => {
    if (query) {
      fetch(`/api/search?q=${query}`)
        .then(res => res.json())
        .then(setResults);
    }
  }, [query], 300);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <Results items={results} />
    </div>
  );
}
```

---

### Pattern 3: Previous Value Hook

```typescript
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}

// Usage
function Counter() {
  const [count, setCount] = useState(0);
  const previousCount = usePrevious(count);

  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {previousCount}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
}
```

---

### Pattern 4: Interval with Dynamic Delay

```typescript
function useInterval(callback: () => void, delay: number | null) {
  const savedCallback = useRef(callback);

  // Update callback ref
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  // Set up interval
  useEffect(() => {
    if (delay === null) return;

    const interval = setInterval(() => {
      savedCallback.current();
    }, delay);

    return () => clearInterval(interval);
  }, [delay]);
}

// Usage
function Timer() {
  const [count, setCount] = useState(0);
  const [delay, setDelay] = useState(1000);

  useInterval(() => {
    setCount(c => c + 1);
  }, delay);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setDelay(delay === 1000 ? 100 : 1000)}>
        Toggle Speed
      </button>
    </div>
  );
}
```

---

## 🏗️ 7. Real-World Examples

### Example 1: WebSocket Connection

```typescript
function useChatMessages(roomId: string) {
  const [messages, setMessages] = useState<Message[]>([]);

  useEffect(() => {
    const ws = new WebSocket(`wss://api.example.com/chat/${roomId}`);

    ws.onopen = () => {
      console.log("Connected to room:", roomId);
    };

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      setMessages((prev) => [...prev, message]);
    };

    ws.onerror = (error) => {
      console.error("WebSocket error:", error);
    };

    ws.onclose = () => {
      console.log("Disconnected from room:", roomId);
    };

    // Cleanup
    return () => {
      ws.close();
    };
  }, [roomId]);

  return messages;
}
```

---

### Example 2: Geolocation Tracking

```typescript
function useGeolocation() {
  const [location, setLocation] = useState<GeolocationCoordinates | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!navigator.geolocation) {
      setError("Geolocation not supported");
      return;
    }

    const watchId = navigator.geolocation.watchPosition(
      (position) => {
        setLocation(position.coords);
        setError(null);
      },
      (err) => {
        setError(err.message);
      },
    );

    return () => {
      navigator.geolocation.clearWatch(watchId);
    };
  }, []);

  return { location, error };
}
```

---

## ❓ 8. Interview Questions

### Q1: When does useEffect cleanup run?

**Answer:**

Cleanup runs in three scenarios:

1. **Before effect re-runs** (dependency changed)
2. **On component unmount**
3. **Never runs if component never unmounts and deps never change**

```jsx
useEffect(() => {
  console.log("Effect");

  return () => {
    console.log("Cleanup");
  };
}, [dep]);

// Lifecycle:
// Mount: 'Effect'
// Update (dep changed): 'Cleanup' → 'Effect'
// Unmount: 'Cleanup'
```

---

### Q2: What's the difference between useEffect and useLayoutEffect?

**Answer:**

| useEffect                    | useLayoutEffect               |
| ---------------------------- | ----------------------------- |
| Runs **after** browser paint | Runs **before** browser paint |
| Async (non-blocking)         | Sync (blocking)               |
| Most effects                 | Visual changes only           |

**When to use useLayoutEffect:**

```jsx
// ❌ useEffect: Flicker (renders twice)
useEffect(() => {
  const box = ref.current;
  box.style.transform = "translateX(100px)";
}, []);

// ✅ useLayoutEffect: No flicker
useLayoutEffect(() => {
  const box = ref.current;
  box.style.transform = "translateX(100px)";
}, []);
```

**Rule:** Use `useEffect` by default, `useLayoutEffect` only if you see visual flicker.

---

### Q3: How do you handle async in useEffect?

**Answer:**

**Can't make useEffect async directly:**

```jsx
// ❌ WRONG
useEffect(async () => {
  // Error!
  const data = await fetchData();
}, []);
```

**Solutions:**

**1. Create async function inside:**

```jsx
useEffect(() => {
  async function fetchData() {
    const data = await fetch("/api/data");
    setData(data);
  }

  fetchData();
}, []);
```

**2. Use IIFE:**

```jsx
useEffect(() => {
  (async () => {
    const data = await fetch("/api/data");
    setData(data);
  })();
}, []);
```

**3. Use .then():**

```jsx
useEffect(() => {
  fetch("/api/data")
    .then((res) => res.json())
    .then(setData);
}, []);
```

**4. With cleanup (AbortController):**

```jsx
useEffect(() => {
  const controller = new AbortController();

  (async () => {
    try {
      const res = await fetch("/api/data", { signal: controller.signal });
      const data = await res.json();
      setData(data);
    } catch (error) {
      if (error.name !== "AbortError") {
        console.error(error);
      }
    }
  })();

  return () => controller.abort();
}, []);
```

---

## 🎯 Summary

**useEffect execution:**

- Render → Paint → Effect → Cleanup (on re-run/unmount)

**Key rules:**

- ✅ Always clean up (event listeners, timers, subscriptions)
- ✅ Declare all dependencies
- ✅ Use functional updates to avoid stale closures
- ✅ Handle race conditions (AbortController, ignore flag)
- ✅ Separate concerns (multiple effects)

**Common pitfalls:**

- ❌ Stale closures (missing deps)
- ❌ Memory leaks (missing cleanup)
- ❌ Race conditions (async updates)
- ❌ Infinite loops (object/function deps)

Master useEffect to build reliable React apps! ⚡
