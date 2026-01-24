# Pure Functions

> **Predictable, testable functions with no side effects**

---

## 🎯 Overview

Pure functions are the foundation of functional programming. A pure function always returns the same output for the same input and has no side effects, making code predictable, testable, and easier to reason about.

---

## 📚 Core Concepts

### **Concept 1: Definition of Pure Functions**

A pure function must satisfy two conditions:

1. **Deterministic** - Same input always produces same output
2. **No side effects** - Doesn't modify external state

```javascript
// ✅ Pure function
function add(a, b) {
  return a + b;
}

add(2, 3); // Always returns 5
add(2, 3); // Always returns 5

// ❌ Impure - uses external state
let tax = 0.1;
function calculateTotal(price) {
  return price + price * tax; // Depends on external variable
}

// ✅ Pure version
function calculateTotal(price, tax) {
  return price + price * tax; // All inputs are parameters
}

// ❌ Impure - modifies external state
let count = 0;
function increment() {
  count++; // Side effect: modifies external variable
  return count;
}

// ✅ Pure version
function increment(count) {
  return count + 1; // Returns new value, doesn't modify input
}
```

### **Concept 2: Side Effects**

Side effects are any interaction with the outside world:

```javascript
// Side effects to avoid:

// ❌ Modifying global/external variables
let globalState = [];
function addItem(item) {
  globalState.push(item); // Side effect
}

// ❌ Mutating input arguments
function addProperty(obj, key, value) {
  obj[key] = value; // Side effect: mutates input
  return obj;
}

// ❌ I/O operations
function readConfig() {
  return fs.readFileSync("config.json"); // Side effect: file I/O
}

// ❌ DOM manipulation
function updateUI(text) {
  document.getElementById("output").textContent = text; // Side effect
}

// ❌ API calls
function fetchUser(id) {
  return fetch(`/api/users/${id}`); // Side effect: network I/O
}

// ❌ Random numbers
function getRandomId() {
  return Math.random(); // Non-deterministic
}

// ❌ Current date/time
function isExpired(expiryDate) {
  return Date.now() > expiryDate; // Non-deterministic
}
```

**Pure alternatives:**

```javascript
// ✅ Pure - return new state
function addItem(array, item) {
  return [...array, item];
}

// ✅ Pure - return new object
function addProperty(obj, key, value) {
  return { ...obj, [key]: value };
}

// ✅ Pure - pass dependencies as parameters
function isExpired(expiryDate, currentDate) {
  return currentDate > expiryDate;
}
```

### **Concept 3: Benefits of Pure Functions**

**1. Testability:**

```javascript
// Pure function - easy to test
function multiply(a, b) {
  return a * b;
}

// No setup needed, no mocking required
test("multiply", () => {
  expect(multiply(2, 3)).toBe(6);
  expect(multiply(0, 5)).toBe(0);
});
```

**2. Predictability:**

```javascript
// Always know what a function does
const result1 = add(2, 3); // 5
const result2 = add(2, 3); // 5 (always)
```

**3. Caching/Memoization:**

```javascript
// Pure functions can be memoized
function memoize(fn) {
  const cache = new Map();
  return function (...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key);
    }
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

const expensiveCalculation = memoize((n) => {
  // Complex calculation
  return n * n;
});
```

**4. Parallelization:**

```javascript
// Pure functions can run in parallel safely
const results = await Promise.all(items.map((item) => processItem(item)));
```

---

## 🔥 Practical Examples

### **Example 1: Array Operations**

```javascript
// ❌ Impure - mutates array
function removeItem(array, index) {
  array.splice(index, 1);
  return array;
}

const items = [1, 2, 3, 4];
removeItem(items, 2);
console.log(items); // [1, 2, 4] - original mutated

// ✅ Pure - returns new array
function removeItem(array, index) {
  return [...array.slice(0, index), ...array.slice(index + 1)];
}

const items = [1, 2, 3, 4];
const newItems = removeItem(items, 2);
console.log(items); // [1, 2, 3, 4] - original unchanged
console.log(newItems); // [1, 2, 4] - new array
```

### **Example 2: Object Updates**

```javascript
// ❌ Impure - mutates object
function updateUser(user, updates) {
  Object.assign(user, updates);
  return user;
}

// ✅ Pure - returns new object
function updateUser(user, updates) {
  return { ...user, ...updates };
}

// ✅ Pure - nested updates
function updateUserAddress(user, newAddress) {
  return {
    ...user,
    address: {
      ...user.address,
      ...newAddress,
    },
  };
}
```

### **Example 3: State Management**

```javascript
// ❌ Impure reducer
function reducer(state, action) {
  switch (action.type) {
    case "ADD_TODO":
      state.todos.push(action.todo); // Mutates state
      return state;
    default:
      return state;
  }
}

// ✅ Pure reducer
function reducer(state, action) {
  switch (action.type) {
    case "ADD_TODO":
      return {
        ...state,
        todos: [...state.todos, action.todo],
      };
    case "REMOVE_TODO":
      return {
        ...state,
        todos: state.todos.filter((todo) => todo.id !== action.id),
      };
    default:
      return state;
  }
}
```

### **Example 4: Dependency Injection**

```javascript
// ❌ Impure - hidden dependencies
function saveUser(user) {
  const db = getDatabase(); // Hidden dependency
  return db.save(user);
}

// ✅ Pure - explicit dependencies
function saveUser(db, user) {
  return db.save(user);
}

// Usage
const db = getDatabase();
saveUser(db, user);
```

---

## 🏗️ Best Practices

1. **Avoid mutating inputs** - Return new values instead

   ```javascript
   // ✅ Good
   function addItem(arr, item) {
     return [...arr, item];
   }
   ```

2. **Make dependencies explicit** - Pass as parameters

   ```javascript
   // ✅ Good
   function calculate(value, config) {
     return value * config.multiplier;
   }
   ```

3. **Keep functions small and focused** - One responsibility

   ```javascript
   // ✅ Good
   function calculateTax(price, rate) {
     return price * rate;
   }

   function calculateTotal(price, taxRate) {
     return price + calculateTax(price, taxRate);
   }
   ```

4. **Isolate side effects** - Keep impure code at boundaries

   ```javascript
   // Pure core
   function processData(data) {
     return data.map(transform).filter(validate);
   }

   // Impure boundary
   async function handleRequest() {
     const data = await fetchData(); // Impure: I/O
     const processed = processData(data); // Pure
     await saveData(processed); // Impure: I/O
   }
   ```

---

## 🧪 Interview Questions

### **Q1: What is a pure function?**

**Answer:** A pure function:

1. Always returns the same output for the same input (deterministic)
2. Has no side effects (doesn't modify external state)

Benefits: Easier to test, reason about, cache, and parallelize.

### **Q2: What are side effects?**

**Answer:** Side effects are any interaction with the outside world:

- Modifying global/external variables
- Mutating input arguments
- I/O operations (file, network, database)
- DOM manipulation
- Logging to console
- Random number generation
- Current date/time access

### **Q3: How do you handle side effects in functional programming?**

**Answer:**

1. **Isolate impure code** - Keep side effects at application boundaries
2. **Dependency injection** - Pass dependencies as parameters
3. **Return new values** - Don't mutate inputs
4. **Use effect management** - Libraries like Redux, RxJS for controlled side effects

---

## 📚 Key Takeaways

- Pure functions are deterministic and have no side effects
- Same input always produces same output
- Never mutate inputs - return new values
- Benefits: testability, predictability, cacheability, parallelization
- Isolate side effects at application boundaries
- Make dependencies explicit through parameters
- Pure functions are the foundation of functional programming

---

**Master pure functions for production-ready code!**
