# Advanced Closures in JavaScript

> **Master currying, partial application, and memoization using closures**

---

## 🎯 Beyond Basic Closures

While basic closures preserve outer scope, advanced closure patterns enable powerful functional programming techniques used in production codebases.

---

## 🔥 Currying

**Currying** transforms a function with multiple arguments into a sequence of functions each taking a single argument.

### **Basic Currying**

```javascript
// Regular function
function add(a, b, c) {
  return a + b + c;
}

// Curried version
function curriedAdd(a) {
  return function (b) {
    return function (c) {
      return a + b + c;
    };
  };
}

console.log(add(1, 2, 3)); // 6
console.log(curriedAdd(1)(2)(3)); // 6
```

### **Generic Curry Function**

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    } else {
      return function (...moreArgs) {
        return curried.apply(this, args.concat(moreArgs));
      };
    }
  };
}

// Usage
function add(a, b, c) {
  return a + b + c;
}

const curriedAdd = curry(add);

console.log(curriedAdd(1)(2)(3)); // 6
console.log(curriedAdd(1, 2)(3)); // 6
console.log(curriedAdd(1)(2, 3)); // 6
```

### **Real-World Currying**

```javascript
// API request builder
const makeRequest = curry((method, url, data) => {
  return fetch(url, {
    method,
    body: JSON.stringify(data),
    headers: { "Content-Type": "application/json" },
  });
});

const get = makeRequest("GET");
const post = makeRequest("POST");
const postToUsers = post("/api/users");

// Use it
postToUsers({ name: "Alice", email: "alice@example.com" });
```

---

## 🎨 Partial Application

**Partial application** fixes some arguments of a function and returns a new function that accepts the remaining arguments.

### **Basic Partial Application**

```javascript
function partial(fn, ...fixedArgs) {
  return function (...remainingArgs) {
    return fn(...fixedArgs, ...remainingArgs);
  };
}

function greet(greeting, name, punctuation) {
  return `${greeting}, ${name}${punctuation}`;
}

const sayHello = partial(greet, "Hello");
console.log(sayHello("Alice", "!")); // "Hello, Alice!"

const greetAlice = partial(greet, "Hello", "Alice");
console.log(greetAlice("!!!")); // "Hello, Alice!!!"
```

### **Currying vs Partial Application**

```javascript
// Currying: one argument at a time
const curriedAdd = (a) => (b) => (c) => a + b + c;
curriedAdd(1)(2)(3); // Must call with one arg each time

// Partial Application: any number of arguments
const partialAdd = partial((a, b, c) => a + b + c, 1);
partialAdd(2, 3); // Can provide multiple args
```

### **Practical Use Case: Event Handlers**

```javascript
function handleEvent(eventType, elementId, callback) {
  document.getElementById(elementId).addEventListener(eventType, callback);
}

// Create specialized handlers
const onClick = partial(handleEvent, "click");
const onClickSubmit = partial(handleEvent, "click", "submitBtn");

// Use them
onClick("myButton", () => console.log("Button clicked"));
onClickSubmit(() => console.log("Submit clicked"));
```

---

## 🚀 Memoization

**Memoization** caches function results to avoid redundant calculations.

### **Basic Memoization**

```javascript
function memoize(fn) {
  const cache = {};

  return function (...args) {
    const key = JSON.stringify(args);

    if (key in cache) {
      console.log("Returning cached result");
      return cache[key];
    }

    console.log("Computing result");
    const result = fn(...args);
    cache[key] = result;
    return result;
  };
}

// Expensive function
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

const memoizedFib = memoize(fibonacci);
console.log(memoizedFib(40)); // Slow first time
console.log(memoizedFib(40)); // Instant second time
```

### **Advanced Memoization with WeakMap**

```javascript
function memoizeWithWeakMap(fn) {
  const cache = new WeakMap();

  return function (obj) {
    if (cache.has(obj)) {
      return cache.get(obj);
    }

    const result = fn(obj);
    cache.set(obj, result);
    return result;
  };
}

// Usage with objects
const processData = memoizeWithWeakMap((data) => {
  // Expensive operation
  return data.items.map((item) => item * 2);
});
```

### **Memoization with TTL (Time To Live)**

```javascript
function memoizeWithTTL(fn, ttl = 5000) {
  const cache = new Map();

  return function (...args) {
    const key = JSON.stringify(args);
    const cached = cache.get(key);

    if (cached && Date.now() - cached.timestamp < ttl) {
      return cached.value;
    }

    const result = fn(...args);
    cache.set(key, {
      value: result,
      timestamp: Date.now(),
    });

    return result;
  };
}

// API call with caching
const fetchUser = memoizeWithTTL(
  async (userId) => {
    const response = await fetch(`/api/users/${userId}`);
    return response.json();
  },
  10000, // Cache for 10 seconds
);
```

---

## 🏗️ Function Composition with Closures

### **Compose Function**

```javascript
const compose =
  (...fns) =>
  (x) =>
    fns.reduceRight((acc, fn) => fn(acc), x);

const pipe =
  (...fns) =>
  (x) =>
    fns.reduce((acc, fn) => fn(acc), x);

// Example functions
const addOne = (x) => x + 1;
const double = (x) => x * 2;
const square = (x) => x * x;

const composed = compose(square, double, addOne);
console.log(composed(3)); // square(double(addOne(3))) = square(double(4)) = square(8) = 64

const piped = pipe(addOne, double, square);
console.log(piped(3)); // square(double(addOne(3))) = 64
```

### **Real-World Composition**

```javascript
// Data transformation pipeline
const trim = (str) => str.trim();
const lowercase = (str) => str.toLowerCase();
const removeSpaces = (str) => str.replace(/\s+/g, "-");

const slugify = pipe(trim, lowercase, removeSpaces);

console.log(slugify("  Hello World  ")); // "hello-world"
```

---

## 💡 Advanced Patterns

### **1. Function Factories**

```javascript
function createMultiplier(multiplier) {
  return function (number) {
    return number * multiplier;
  };
}

const double = createMultiplier(2);
const triple = createMultiplier(3);

console.log(double(5)); // 10
console.log(triple(5)); // 15
```

### **2. Private Variables and Methods**

```javascript
function createBankAccount(initialBalance) {
  let balance = initialBalance;
  const transactions = [];

  // Private method
  function recordTransaction(type, amount) {
    transactions.push({
      type,
      amount,
      date: new Date(),
      balance: balance,
    });
  }

  return {
    deposit(amount) {
      balance += amount;
      recordTransaction("deposit", amount);
      return balance;
    },

    withdraw(amount) {
      if (amount > balance) {
        throw new Error("Insufficient funds");
      }
      balance -= amount;
      recordTransaction("withdrawal", amount);
      return balance;
    },

    getBalance() {
      return balance;
    },

    getTransactions() {
      return [...transactions]; // Return copy
    },
  };
}

const account = createBankAccount(1000);
account.deposit(500);
account.withdraw(200);
console.log(account.getBalance()); // 1300
console.log(account.getTransactions()); // Full history
```

### **3. Once Function**

```javascript
function once(fn) {
  let called = false;
  let result;

  return function (...args) {
    if (!called) {
      called = true;
      result = fn(...args);
    }
    return result;
  };
}

const initialize = once(() => {
  console.log("Initializing...");
  return { initialized: true };
});

initialize(); // Logs: "Initializing..."
initialize(); // Does nothing
initialize(); // Does nothing
```

### **4. Debounce and Throttle**

```javascript
function debounce(fn, delay) {
  let timeoutId;

  return function (...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
}

function throttle(fn, limit) {
  let inThrottle;

  return function (...args) {
    if (!inThrottle) {
      fn(...args);
      inThrottle = true;
      setTimeout(() => (inThrottle = false), limit);
    }
  };
}

// Usage
const handleSearch = debounce((query) => {
  console.log("Searching for:", query);
}, 500);

const handleScroll = throttle(() => {
  console.log("Scroll event");
}, 1000);
```

---

## 🎯 React Hooks Analogy

Closures are fundamental to React hooks:

```javascript
function useState(initialValue) {
  let state = initialValue;

  function setState(newValue) {
    state = newValue;
    // Trigger re-render
  }

  return [state, setState];
}

function useEffect(callback, dependencies) {
  let previousDeps;

  return function () {
    const hasChanged =
      !previousDeps || dependencies.some((dep, i) => dep !== previousDeps[i]);

    if (hasChanged) {
      callback();
      previousDeps = dependencies;
    }
  };
}
```

---

## 🧪 Interview Questions

### **Q1: Implement a `memoize` function that works with multiple arguments**

```javascript
function memoize(fn) {
  // Your implementation
}

const add = (a, b, c) => a + b + c;
const memoizedAdd = memoize(add);

console.log(memoizedAdd(1, 2, 3)); // Computes
console.log(memoizedAdd(1, 2, 3)); // Returns cached
```

### **Q2: What's the difference between currying and partial application?**

**Answer:**

- **Currying**: Transforms `f(a, b, c)` into `f(a)(b)(c)` - always one argument at a time
- **Partial Application**: Fixes some arguments, returns function accepting the rest - any number of arguments

### **Q3: Implement a `curry` function that works for any number of arguments**

```javascript
function curry(fn) {
  // Your implementation
}
```

---

## 🎯 Best Practices

1. **Use currying for reusable configurations**
2. **Apply memoization to expensive, pure functions**
3. **Be careful with cache size** - implement cache limits if needed
4. **Use WeakMap for object caching** to allow garbage collection
5. **Debounce user inputs** to improve performance
6. **Throttle scroll/resize events** to prevent performance issues
7. **Document closure usage** - they can be harder to understand

---

## 📚 Key Takeaways

- **Currying** transforms multi-arg functions into unary function chains
- **Partial application** fixes some arguments, creating specialized functions
- **Memoization** caches results to optimize expensive computations
- **Closures enable data privacy** and encapsulation patterns
- **Function composition** creates complex behavior from simple functions
- **Debounce/throttle** are essential for performance optimization

---

**Master these patterns to write cleaner, more functional JavaScript code!**
