# 🔄 Closures & Scope Chain

> "Closures are functions that remember their birthplace." This single concept powers most advanced JavaScript patterns: callbacks, modules, currying, memoization, and React hooks.

---

## 📖 1. Concept Explanation

### What is a Closure?

A **closure** is created when a function retains access to variables from its outer (enclosing) lexical scope, even after the outer function has finished executing.

```javascript
function outer() {
  const secret = "data";

  return function inner() {
    console.log(secret); // Has access to 'secret'
  };
}

const closure = outer();
// outer() has returned, its execution context is gone
// But 'secret' is still accessible!

closure(); // 'data'
```

### The Three Ingredients of Closures

1. **Nested Function** - Inner function defined inside outer function
2. **Variable Reference** - Inner function references outer function's variable
3. **Function Return/Export** - Inner function escapes its parent scope

### Lexical Scoping

Closures work because of **lexical scoping**: functions are executed using the scope chain that was in effect when they were **defined**, not when they're **called**.

```javascript
const x = 10;

function foo() {
  console.log(x); // Looks for 'x' in lexical scope
}

function bar() {
  const x = 20;
  foo(); // What does this log?
}

bar(); // Logs 10 (not 20!)
// foo's lexical scope chain: foo → global
// NOT foo → bar → global
```

---

## 🧠 2. Why It Matters in Real Projects

### Production Use Cases

**1. Data Privacy (Encapsulation):**

```javascript
// Without closures: public data
const user = {
  password: "secret123", // ❌ Anyone can access
};

// With closures: private data
function createUser(password) {
  let _password = password; // Private

  return {
    checkPassword(input) {
      return input === _password;
    },
    changePassword(oldPass, newPass) {
      if (oldPass === _password) {
        _password = newPass;
        return true;
      }
      return false;
    },
  };
}

const user = createUser("secret123");
user._password; // undefined (private!)
user.checkPassword("secret123"); // true
```

**2. React Hooks:**

```javascript
// useState is implemented with closures!
function useState(initialValue) {
  let value = initialValue; // Closed over

  function setState(newValue) {
    value = newValue;
    triggerReRender(); // Simplified
  }

  return [value, setState];
}
```

**3. Event Handlers:**

```javascript
function setupButton(id) {
  const button = document.getElementById(id);

  let clickCount = 0; // Closed over by handler

  button.addEventListener("click", function () {
    clickCount++;
    console.log(`Clicked ${clickCount} times`);
  });
}
```

**4. Module Pattern:**

```javascript
const ShoppingCart = (function () {
  const items = []; // Private

  return {
    addItem(item) {
      items.push(item);
    },
    getTotal() {
      return items.reduce((sum, item) => sum + item.price, 0);
    },
    getItemCount() {
      return items.length;
    },
  };
})();
```

### Performance Impact

**Good:**

- Enables powerful patterns (memoization, currying)
- Alternative to global variables

**Bad:**

- Memory overhead (keeps scope alive)
- Potential memory leaks if not managed
- Garbage collection complexity

---

## ⚙️ 3. Internal Working

### How V8 Handles Closures

When a closure is created, V8 doesn't copy all outer variables. It creates a **context** object containing only the variables referenced by the inner function.

```javascript
function outer() {
  const used = "kept"; // Referenced by inner
  const notUsed = "freed"; // NOT referenced
  let counter = 0; // Referenced by inner

  return function inner() {
    counter++;
    return used + counter;
  };
}

const closure = outer();
```

**Memory structure:**

```
closure (function object)
  └─ [[Scopes]]
       └─ Closure (outer)
            ├─ used: 'kept'
            └─ counter: 0
       └─ Global

// 'notUsed' is NOT kept in memory!
```

### Scope Chain Resolution

```javascript
const global = "global";

function level1() {
  const var1 = "level1";

  function level2() {
    const var2 = "level2";

    function level3() {
      const var3 = "level3";
      console.log(var3); // Found in level3
      console.log(var2); // Found in level2 (closure)
      console.log(var1); // Found in level1 (closure)
      console.log(global); // Found in global
    }

    return level3;
  }

  return level2();
}

const fn = level1();
fn(); // All variables accessible through closures
```

**Scope chain for `level3`:**

```
level3 → level2 → level1 → global → ReferenceError
```

### Memory Lifecycle

```javascript
function createCounter() {
  let count = 0; // [1] Allocated in outer's context

  return function increment() {
    count++; // [2] Closure keeps 'count' alive
    return count;
  };
}

const counter1 = createCounter(); // [3] outer returns, but 'count' persists
counter1(); // 1
counter1(); // 2

const counter2 = createCounter(); // [4] New closure, independent 'count'
counter2(); // 1

counter1 = null; // [5] Closure released, 'count' can be GC'd
```

---

## ✅ 4. Best Practices

### DO ✅

```javascript
// 1. Use closures for private data
function createBankAccount(initialBalance) {
  let balance = initialBalance; // Private

  return {
    deposit(amount) {
      balance += amount;
      return balance;
    },
    getBalance() {
      return balance;
    },
  };
}

// 2. Avoid capturing unnecessary variables
function processData(data) {
  const hugeArray = Array(1000000).fill(0);
  const small = hugeArray.length; // Extract only what you need

  return function () {
    return small; // Only closes over 'small', not 'hugeArray'
  };
}

// 3. Use arrow functions for lexical 'this'
class Timer {
  constructor() {
    this.seconds = 0;
  }

  start() {
    setInterval(() => {
      this.seconds++; // Closure over 'this'
    }, 1000);
  }
}

// 4. Null out closures to allow GC
let heavyClosure = createHeavyClosure();
// ... use it ...
heavyClosure = null; // Allow GC

// 5. Use block scope to limit closure capture
{
  const tempData = fetchData();
  const processed = process(tempData);
  // tempData can be GC'd here

  return function () {
    return processed; // Only closes over 'processed'
  };
}
```

### DON'T ❌

```javascript
// 1. Don't create closures in loops unnecessarily
// ❌ BAD
for (let i = 0; i < 1000; i++) {
  buttons[i].onclick = function () {
    console.log("Clicked");
  }; // 1000 separate closures!
}

// ✅ GOOD
const handleClick = function () {
  console.log("Clicked");
};
for (let i = 0; i < 1000; i++) {
  buttons[i].onclick = handleClick; // Single function reference
}

// 2. Don't capture huge objects when you need one property
// ❌ BAD
function process(user) {
  // Keeps entire 'user' object in memory
  return function () {
    return user.name;
  };
}

// ✅ GOOD
function process(user) {
  const name = user.name; // Extract only what's needed
  return function () {
    return name;
  };
}

// 3. Don't forget to clean up event listeners
// ❌ BAD (memory leak in SPAs)
function setupComponent() {
  const data = fetchData();

  window.addEventListener("resize", function () {
    renderWithData(data); // Closure keeps 'data' alive
  });
  // Component unmounts, but listener (and 'data') remain!
}

// ✅ GOOD
function setupComponent() {
  const data = fetchData();

  const handler = function () {
    renderWithData(data);
  };

  window.addEventListener("resize", handler);

  return function cleanup() {
    window.removeEventListener("resize", handler);
  };
}
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Unintentional Memory Leaks

```javascript
// ❌ MEMORY LEAK
function attachHandler() {
  const hugeData = new Array(1000000).fill("data");

  button.addEventListener("click", function () {
    console.log("clicked");
    // Closure captures 'hugeData' even though unused
  });
}

// Each call creates new closure, hugeData never released
for (let i = 0; i < 100; i++) {
  attachHandler(); // 100MB+ memory leak!
}

// ✅ FIX 1: Don't capture unused variables
function attachHandler() {
  button.addEventListener("click", function () {
    const hugeData = new Array(1000000).fill("data");
    console.log("clicked");
    // hugeData can be GC'd after function execution
  });
}

// ✅ FIX 2: Set to null when done
function attachHandler() {
  let hugeData = new Array(1000000).fill("data");

  button.addEventListener("click", function () {
    if (hugeData) {
      process(hugeData);
      hugeData = null; // Allow GC
    }
  });
}
```

### Mistake #2: Closure in Loop (Classic)

```javascript
// ❌ WRONG
const functions = [];
for (var i = 0; i < 3; i++) {
  functions.push(function () {
    console.log(i);
  });
}

functions[0](); // 3 (not 0!)
functions[1](); // 3 (not 1!)
functions[2](); // 3 (not 2!)

// Why? All closures share same 'i' (var is function-scoped)

// ✅ FIX 1: Use let (block-scoped)
const functions = [];
for (let i = 0; i < 3; i++) {
  functions.push(function () {
    console.log(i);
  });
}

functions[0](); // 0 ✅
functions[1](); // 1 ✅
functions[2](); // 2 ✅

// ✅ FIX 2: IIFE (old way)
for (var i = 0; i < 3; i++) {
  (function (j) {
    functions.push(function () {
      console.log(j);
    });
  })(i);
}
```

### Mistake #3: Modifying Closed-Over Variables

```javascript
// ❌ CONFUSING
function createCounter() {
  let count = 0;

  return {
    increment() {
      return ++count;
    },
    decrement() {
      return --count;
    },
  };
}

const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2

// Someone might expect independent counts, but they share 'count'
```

### Mistake #4: React Component Memory Leaks

```javascript
// ❌ MEMORY LEAK IN REACT
function DataFetcher() {
  const [data, setData] = useState(null);

  useEffect(() => {
    const interval = setInterval(() => {
      fetch("/api/data")
        .then((res) => res.json())
        .then(setData); // Closure captures 'setData'
    }, 1000);

    // ❌ FORGOT CLEANUP
  }, []);

  return <div>{data}</div>;
}

// Component unmounts, but interval keeps running!

// ✅ FIX: Cleanup function
function DataFetcher() {
  const [data, setData] = useState(null);

  useEffect(() => {
    const interval = setInterval(() => {
      fetch("/api/data")
        .then((res) => res.json())
        .then(setData);
    }, 1000);

    return () => clearInterval(interval); // Cleanup
  }, []);

  return <div>{data}</div>;
}
```

---

## 🔐 6. Security Considerations

### Private Data Protection

```javascript
// ❌ EXPOSED: Anyone can modify
const API_CONFIG = {
  apiKey: "secret123",
  endpoint: "https://api.example.com",
};

// Attacker can do:
API_CONFIG.apiKey = "hacked";

// ✅ PROTECTED: Closure hides data
const createAPIClient = (function () {
  const apiKey = "secret123"; // Private

  return {
    makeRequest(endpoint) {
      return fetch(endpoint, {
        headers: { Authorization: `Bearer ${apiKey}` },
      });
    },
  };
})();

// apiKey is not accessible from outside
```

### Avoiding Information Leaks

```javascript
// ❌ LEAKY: Error messages expose closure data
function createValidator(secret) {
  return function validate(input) {
    if (input !== secret) {
      throw new Error(`Expected ${secret}, got ${input}`); // ❌ Leaks 'secret'!
    }
  };
}

// ✅ SAFE: Don't expose sensitive data
function createValidator(secret) {
  return function validate(input) {
    if (input !== secret) {
      throw new Error("Validation failed"); // ✅ Generic message
    }
  };
}
```

---

## 🚀 7. Performance Optimization

### 1. Memoization with Closures

```javascript
// Expensive function
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2); // Exponential time!
}

// ✅ OPTIMIZED: Memoization with closure
function memoize(fn) {
  const cache = {}; // Closure variable

  return function (...args) {
    const key = JSON.stringify(args);
    if (key in cache) {
      return cache[key]; // Cached result
    }
    const result = fn.apply(this, args);
    cache[key] = result;
    return result;
  };
}

const fastFib = memoize(fibonacci);
fastFib(40); // Slow first call
fastFib(40); // Instant (cached)
```

### 2. Partial Application & Currying

```javascript
// Currying with closures
function multiply(a) {
  return function (b) {
    return a * b;
  };
}

const multiplyBy2 = multiply(2); // Closure over 'a = 2'
multiplyBy2(5); // 10
multiplyBy2(3); // 6

// Real-world: Specialized API clients
function createAPIClient(baseURL) {
  return function (endpoint) {
    return function (options) {
      return fetch(baseURL + endpoint, options);
    };
  };
}

const apiClient = createAPIClient("https://api.example.com");
const usersAPI = apiClient("/users");
usersAPI({ method: "GET" });
```

### 3. Minimize Closure Scope

```javascript
// ❌ SLOW: Captures entire large object
function processUser(user) {
  return function () {
    return user.name; // Keeps entire 'user' in memory
  };
}

// ✅ FAST: Only capture what's needed
function processUser(user) {
  const name = user.name;
  return function () {
    return name; // Only 'name' in closure
  };
}

// Benchmark: 10,000 closures
// Bad version: ~50MB memory
// Good version: ~5MB memory
```

---

## 🧪 8. Code Examples

### Example 1: Counter with Closure

```javascript
function createCounter(initial = 0) {
  let count = initial;

  return {
    increment() {
      return ++count;
    },
    decrement() {
      return --count;
    },
    reset() {
      count = initial;
      return count;
    },
    getValue() {
      return count;
    },
  };
}

const counter = createCounter(10);
console.log(counter.increment()); // 11
console.log(counter.increment()); // 12
console.log(counter.decrement()); // 11
console.log(counter.reset()); // 10
console.log(counter.count); // undefined (private)
```

### Example 2: Once Function

```javascript
function once(fn) {
  let called = false;
  let result;

  return function (...args) {
    if (!called) {
      called = true;
      result = fn.apply(this, args);
    }
    return result;
  };
}

const initialize = once(() => {
  console.log("Initialized!");
  return { data: "loaded" };
});

initialize(); // Logs "Initialized!", returns { data: 'loaded' }
initialize(); // Returns { data: 'loaded' } (no log)
initialize(); // Returns { data: 'loaded' } (no log)
```

### Example 3: Private Variables Pattern

```javascript
class BankAccount {
  #balance; // Private field (ES2022+)

  constructor(initialBalance) {
    this.#balance = initialBalance;
  }

  deposit(amount) {
    if (amount > 0) {
      this.#balance += amount;
    }
    return this.#balance;
  }

  withdraw(amount) {
    if (amount > 0 && amount <= this.#balance) {
      this.#balance -= amount;
      return amount;
    }
    return 0;
  }

  getBalance() {
    return this.#balance;
  }
}

// Closure-based alternative (pre-ES2022)
function createBankAccount(initialBalance) {
  let balance = initialBalance; // Private via closure

  return {
    deposit(amount) {
      if (amount > 0) {
        balance += amount;
      }
      return balance;
    },
    withdraw(amount) {
      if (amount > 0 && amount <= balance) {
        balance -= amount;
        return amount;
      }
      return 0;
    },
    getBalance() {
      return balance;
    },
  };
}
```

### Example 4: Module Pattern

```javascript
const UserModule = (function () {
  // Private data
  const users = [];
  let currentUser = null;

  // Private functions
  function validateUser(user) {
    return user && user.email && user.name;
  }

  function hashPassword(password) {
    // Simplified
    return btoa(password);
  }

  // Public API
  return {
    register(email, name, password) {
      const user = {
        id: Date.now(),
        email,
        name,
        password: hashPassword(password),
      };

      if (validateUser(user)) {
        users.push(user);
        return user.id;
      }
      return null;
    },

    login(email, password) {
      const user = users.find(
        (u) => u.email === email && u.password === hashPassword(password),
      );

      if (user) {
        currentUser = user;
        return true;
      }
      return false;
    },

    getCurrentUser() {
      return currentUser ? { ...currentUser, password: undefined } : null;
    },

    logout() {
      currentUser = null;
    },
  };
})();

// Usage
UserModule.register("alice@example.com", "Alice", "password123");
UserModule.login("alice@example.com", "password123");
console.log(UserModule.getCurrentUser()); // { id: ..., email: ..., name: ... }
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: Debounce Implementation

```javascript
function debounce(func, delay) {
  let timeoutId; // Closure variable

  return function (...args) {
    const context = this;

    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => {
      func.apply(context, args);
    }, delay);
  };
}

// Usage: Search input
const searchInput = document.getElementById("search");
const search = debounce(function (e) {
  console.log("Searching for:", e.target.value);
  // API call here
}, 300);

searchInput.addEventListener("input", search);
```

### Use Case 2: React Custom Hook with Closure

```javascript
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });

  // Closure over 'key'
  const setValue = useCallback(
    (value) => {
      try {
        setStoredValue(value);
        window.localStorage.setItem(key, JSON.stringify(value));
      } catch (error) {
        console.error(error);
      }
    },
    [key],
  );

  return [storedValue, setValue];
}

// Usage
function App() {
  const [name, setName] = useLocalStorage("name", "Guest");

  return <input value={name} onChange={(e) => setName(e.target.value)} />;
}
```

### Use Case 3: API Client Factory

```javascript
function createAPIClient(config) {
  const { baseURL, apiKey } = config; // Closure over config

  const cache = new Map(); // Private cache

  return {
    async get(endpoint, options = {}) {
      const cacheKey = `GET:${endpoint}`;

      if (cache.has(cacheKey) && !options.skipCache) {
        return cache.get(cacheKey);
      }

      const response = await fetch(baseURL + endpoint, {
        headers: {
          Authorization: `Bearer ${apiKey}`,
          ...options.headers,
        },
      });

      const data = await response.json();
      cache.set(cacheKey, data);
      return data;
    },

    async post(endpoint, body) {
      return fetch(baseURL + endpoint, {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify(body),
      }).then((r) => r.json());
    },

    clearCache() {
      cache.clear();
    },
  };
}

// Usage
const apiClient = createAPIClient({
  baseURL: "https://api.example.com",
  apiKey: "secret123",
});

await apiClient.get("/users");
await apiClient.post("/users", { name: "Alice" });
```

---

## ❓ 10. Interview Questions

### Q1: What is a closure and how does it work?

**Answer:**

A closure is a function that has access to variables from its outer (enclosing) scope, even after the outer function has finished executing.

**How it works:**

1. When a function is created, it gets a hidden `[[Scopes]]` property
2. This property references the lexical environment where the function was defined
3. When the function executes, it can access variables through this scope chain
4. Even if the outer function returns, the inner function maintains its reference to the outer scope

```javascript
function outer() {
  const secret = "data";

  return function inner() {
    console.log(secret); // Closure
  };
}

const fn = outer();
fn(); // 'data' - outer is gone, but 'secret' accessible
```

**Follow-up:** What's the memory implication?

- Closures keep their outer scope alive, preventing garbage collection
- Can cause memory leaks if not managed properly

---

### Q2: Explain this classic interview question:

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(function () {
    console.log(i);
  }, i * 1000);
}
```

**Answer:**

Output: `3, 3, 3` (after 0s, 1s, 2s)

**Why:**

- `var` is function-scoped (or global), not block-scoped
- All three closures capture the SAME `i` variable
- By the time callbacks execute, the loop has finished and `i = 3`

**Fixes:**

1. **Use `let` (creates new binding per iteration):**

```javascript
for (let i = 0; i < 3; i++) {
  setTimeout(function () {
    console.log(i); // 0, 1, 2
  }, i * 1000);
}
```

2. **IIFE (creates new scope):**

```javascript
for (var i = 0; i < 3; i++) {
  (function (j) {
    setTimeout(function () {
      console.log(j); // 0, 1, 2
    }, j * 1000);
  })(i);
}
```

---

### Q3: How would you implement a function that can only be called once?

**Answer:**

```javascript
function once(fn) {
  let called = false;
  let result;

  return function (...args) {
    if (!called) {
      called = true;
      result = fn.apply(this, args);
    }
    return result;
  };
}

// Usage
const initialize = once(() => {
  console.log("Initialized");
  return { data: "loaded" };
});

initialize(); // Logs "Initialized", returns { data: 'loaded' }
initialize(); // Returns { data: 'loaded' } (no log)
```

**Key points:**

- `called` and `result` are closed over
- First call executes function and caches result
- Subsequent calls return cached result
- Maintains `this` binding with `apply`

---

### Q4: What are the downsides of closures?

**Answer:**

**1. Memory Consumption:**

- Closures keep outer scope alive, preventing GC
- Can lead to memory leaks if not managed

```javascript
function createLeaky() {
  const huge = new Array(1000000).fill("data");

  return function () {
    console.log("leaky");
    // 'huge' kept in memory even though unused
  };
}
```

**2. Performance:**

- Scope chain lookup has cost
- More memory = more GC pressure

**3. Debugging:**

- Private variables not visible in debugger
- Harder to inspect state

**4. Accidental Sharing:**

```javascript
function createHandlers() {
  const handlers = [];
  for (var i = 0; i < 3; i++) {
    handlers.push(() => console.log(i)); // All share same 'i'
  }
  return handlers;
}
```

**Mitigation:**

- Extract only needed variables
- Set closures to `null` when done
- Use weak references where appropriate
- Profile memory usage

---

### Q5: How do React hooks use closures? Can you explain stale closures?

**Answer:**

**React Hooks = Closures:**

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  // This function closes over 'count'
  function handleClick() {
    console.log(count); // Closure
    setCount(count + 1);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

**Stale Closure Problem:**

```javascript
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      console.log(count); // ❌ Always logs 0 (stale closure)
      setCount(count + 1); // ❌ Always sets to 1
    }, 1000);

    return () => clearInterval(interval);
  }, []); // Empty deps = closure over initial 'count'

  return <div>{count}</div>;
}
```

**Why stale:**

- `useEffect` runs once (empty deps)
- Callback closes over `count = 0`
- Future renders have new `count`, but interval still has old closure

**Fixes:**

1. **Functional update:**

```javascript
setCount((prevCount) => prevCount + 1); // ✅ No dependency on 'count'
```

2. **Include in dependencies:**

```javascript
useEffect(() => {
  const interval = setInterval(() => {
    setCount(count + 1);
  }, 1000);

  return () => clearInterval(interval);
}, [count]); // ✅ Recreates interval when 'count' changes
```

3. **useRef (doesn't trigger re-render):**

```javascript
const countRef = useRef(count);
countRef.current = count;

useEffect(() => {
  const interval = setInterval(() => {
    setCount(countRef.current + 1);
  }, 1000);

  return () => clearInterval(interval);
}, []);
```

---

## 🧩 11. Design Patterns

### Pattern 1: Module Pattern

**Purpose:** Encapsulation with private data

```javascript
const Calculator = (function () {
  // Private
  let history = [];

  function log(operation, result) {
    history.push({ operation, result, timestamp: Date.now() });
  }

  // Public API
  return {
    add(a, b) {
      const result = a + b;
      log("add", result);
      return result;
    },

    subtract(a, b) {
      const result = a - b;
      log("subtract", result);
      return result;
    },

    getHistory() {
      return [...history]; // Return copy
    },

    clearHistory() {
      history = [];
    },
  };
})();
```

**Pros:** Encapsulation, clear API  
**Cons:** Memory overhead, no inheritance

---

### Pattern 2: Memoization

```javascript
function memoize(fn, keyFn = JSON.stringify) {
  const cache = new Map();

  return function (...args) {
    const key = keyFn(args);

    if (cache.has(key)) {
      return cache.get(key);
    }

    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// Usage
const expensiveCalc = (n) => {
  console.log("Calculating...");
  return n * 2;
};

const memoized = memoize(expensiveCalc);
memoized(5); // Logs "Calculating...", returns 10
memoized(5); // Returns 10 (no log, cached)
```

---

### Pattern 3: Partial Application

```javascript
function partial(fn, ...fixedArgs) {
  return function (...remainingArgs) {
    return fn.apply(this, [...fixedArgs, ...remainingArgs]);
  };
}

// Usage
function greet(greeting, name) {
  return `${greeting}, ${name}!`;
}

const sayHello = partial(greet, "Hello");
sayHello("Alice"); // "Hello, Alice!"
sayHello("Bob"); // "Hello, Bob!"

const sayHi = partial(greet, "Hi");
sayHi("Charlie"); // "Hi, Charlie!"
```

---

## 🎯 Summary

Closures are the foundation of JavaScript's functional programming capabilities:

- **Definition:** Function + its lexical environment
- **Use cases:** Encapsulation, callbacks, modules, React hooks
- **Memory:** Keeps outer scope alive (can cause leaks)
- **Performance:** Consider closure scope size
- **Patterns:** Module, memoization, currying, partial application

**Interview key points:**

1. Explain lexical scoping
2. Understand memory implications
3. Know common pitfalls (loop + closure)
4. Relate to React hooks (stale closures)
5. Demonstrate with practical examples

**Production checklist:**

- ✅ Extract only needed variables
- ✅ Clean up event listeners
- ✅ Use `useCallback`/`useMemo` in React
- ✅ Profile memory usage
- ✅ Set closures to `null` when done
