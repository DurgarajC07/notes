# 📝 JavaScript Interview Questions (Senior Level)

> Comprehensive collection of JavaScript questions asked in Senior/Staff Frontend Engineer interviews at top tech companies (FAANG+). Focus on internals, trade-offs, and practical application.

---

## 🎯 Part 1: Core JavaScript

### Q1: Explain closures with a real-world example

**Answer:**

A **closure** is a function that remembers variables from its outer scope, even after the outer function has returned.

**Real-world example: Private variables**

```javascript
function createBankAccount(initialBalance) {
  let balance = initialBalance; // Private variable

  return {
    deposit(amount) {
      balance += amount;
      return balance;
    },
    withdraw(amount) {
      if (amount > balance) {
        throw new Error("Insufficient funds");
      }
      balance -= amount;
      return balance;
    },
    getBalance() {
      return balance;
    },
  };
}

const account = createBankAccount(1000);
console.log(account.getBalance()); // 1000
account.deposit(500); // 1500
account.withdraw(200); // 1300

// ✅ balance is private - can't access directly
console.log(account.balance); // undefined
```

**Why closures matter:**

- Data privacy (no direct access to `balance`)
- Factory pattern
- Event handlers
- Callbacks with context

**Follow-up: Memory implications?**

Closures keep outer variables in memory:

```javascript
function heavyOperation() {
  const largeArray = new Array(1000000).fill("data");

  return function () {
    // This closure keeps largeArray in memory!
    console.log(largeArray.length);
  };
}

const fn = heavyOperation();
// largeArray still in memory even after heavyOperation returned
```

---

### Q2: What's the difference between `==` and `===`?

**Answer:**

| `==` (Loose Equality) | `===` (Strict Equality) |
| --------------------- | ----------------------- |
| Type coercion         | No type coercion        |
| `5 == '5'` → `true`   | `5 === '5'` → `false`   |
| Unpredictable         | Predictable             |

**Coercion examples:**

```javascript
// ❌ Confusing with ==
0 == false        // true
'' == false       // true
null == undefined // true
[] == false       // true
[1] == 1          // true

// ✅ Clear with ===
0 === false        // false
'' === false       // false
null === undefined // false
[] === false       // false
[1] === 1          // false
```

**Coercion algorithm (==):**

```javascript
// If types differ:
// 1. null == undefined → true
// 2. String/Boolean → convert to Number
// 3. Object → convert to primitive (valueOf/toString)

'5' == 5
// '5' → Number('5') → 5
// 5 === 5 → true

[] == false
// [] → [].toString() → ''
// '' == false → Number('') === Number(false)
// 0 === 0 → true
```

**Production rule: Always use `===` unless you explicitly need coercion.**

**Exception: Checking for null/undefined:**

```javascript
// ✅ Only case for ==
if (value == null) {
  // true if value is null OR undefined
}

// Equivalent to:
if (value === null || value === undefined) {
  // More explicit but verbose
}
```

---

### Q3: How does the event loop work?

**Answer:**

The event loop coordinates execution between the **call stack**, **microtask queue**, and **task queue**.

**Architecture:**

```
┌─────────────────────┐
│    Call Stack       │ ← Sync code
└─────────────────────┘
          ↓
┌─────────────────────┐
│  Microtask Queue    │ ← Promises (higher priority)
└─────────────────────┘
          ↓
┌─────────────────────┐
│    Task Queue       │ ← setTimeout, I/O (lower priority)
└─────────────────────┘
```

**Execution order:**

```javascript
console.log("1"); // Sync

setTimeout(() => console.log("4"), 0); // Task

Promise.resolve().then(() => console.log("3")); // Microtask

console.log("2"); // Sync

// Output: 1, 2, 3, 4
```

**Algorithm:**

```
1. Execute all synchronous code (call stack)
2. Process ALL microtasks (Promises)
3. Render (if needed)
4. Process ONE task (setTimeout)
5. Repeat
```

**Key insight:** ALL microtasks execute before ANY task!

**Follow-up: What if you create infinite microtasks?**

```javascript
// ❌ Microtask starvation
function loop() {
  Promise.resolve().then(() => {
    console.log("Microtask");
    loop(); // Infinite microtasks!
  });
}

loop();

setTimeout(() => {
  console.log("This never runs!"); // Starved by microtasks
}, 0);
```

---

### Q4: Explain prototypal inheritance

**Answer:**

JavaScript uses **prototype chain** for inheritance, not classes (even ES6 classes are syntactic sugar).

**Prototype chain:**

```javascript
const obj = { a: 1 };

obj.toString(); // Where is toString()?

// Lookup chain:
// 1. obj.toString → not found
// 2. obj.__proto__.toString → Object.prototype.toString → FOUND!
```

**How it works:**

```javascript
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  return `Hi, I'm ${this.name}`;
};

const alice = new Person("Alice");

alice.greet(); // 'Hi, I'm Alice'

// Lookup:
// 1. alice.greet → not found
// 2. alice.__proto__ → Person.prototype.greet → FOUND!
```

**Memory benefit: Methods shared on prototype**

```javascript
// ❌ Memory inefficient
function Person(name) {
  this.name = name;
  this.greet = function () {
    // New function per instance!
    return `Hi, I'm ${this.name}`;
  };
}

// ✅ Memory efficient
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  // Shared across all instances
  return `Hi, I'm ${this.name}`;
};
```

**ES6 classes are syntactic sugar:**

```javascript
class Person {
  constructor(name) {
    this.name = name;
  }

  greet() {
    return `Hi, I'm ${this.name}`;
  }
}

// Equivalent to:
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  return `Hi, I'm ${this.name}`;
};
```

**Follow-up: What's the difference between `__proto__` and `prototype`?**

- `__proto__`: Instance property pointing to prototype
- `prototype`: Constructor property that becomes `__proto__` of instances

```javascript
function Person() {}

const alice = new Person();

alice.__proto__ === Person.prototype; // true
```

---

### Q5: What's `this` binding in JavaScript?

**Answer:**

`this` depends on **how the function is called**, not where it's defined.

**Four binding rules:**

**1. Default binding (global object):**

```javascript
function foo() {
  console.log(this); // Window (or undefined in strict mode)
}

foo();
```

**2. Implicit binding (object.method()):**

```javascript
const obj = {
  name: "Alice",
  greet() {
    console.log(this.name); // 'Alice'
  },
};

obj.greet(); // this = obj
```

**3. Explicit binding (call/apply/bind):**

```javascript
function greet() {
  console.log(this.name);
}

const person = { name: "Alice" };

greet.call(person); // 'Alice' - this = person
greet.apply(person); // 'Alice' - this = person

const boundGreet = greet.bind(person);
boundGreet(); // 'Alice' - this permanently bound
```

**4. `new` binding:**

```javascript
function Person(name) {
  this.name = name; // this = new empty object
}

const alice = new Person("Alice");
// 1. Create empty object
// 2. Set __proto__
// 3. Call Person with this = new object
// 4. Return new object
```

**Arrow functions: Lexical `this`**

```javascript
const obj = {
  name: "Alice",
  greet() {
    // ❌ Regular function: this = global
    setTimeout(function () {
      console.log(this.name); // undefined
    }, 100);

    // ✅ Arrow function: this = obj
    setTimeout(() => {
      console.log(this.name); // 'Alice'
    }, 100);
  },
};

obj.greet();
```

**Common pitfall: Lost binding**

```javascript
const obj = {
  name: "Alice",
  greet() {
    console.log(this.name);
  },
};

const greet = obj.greet;
greet(); // undefined - lost binding!

// Fix: Bind explicitly
const boundGreet = obj.greet.bind(obj);
boundGreet(); // 'Alice'
```

---

## 🎯 Part 2: Async JavaScript

### Q6: What's the difference between Promise and async/await?

**Answer:**

**async/await is syntactic sugar over Promises.**

**Promise chains:**

```javascript
fetch("/api/user")
  .then((res) => res.json())
  .then((user) => fetch(`/api/posts/${user.id}`))
  .then((res) => res.json())
  .then((posts) => console.log(posts))
  .catch((error) => console.error(error));
```

**async/await (cleaner):**

```javascript
async function getPosts() {
  try {
    const userRes = await fetch("/api/user");
    const user = await userRes.json();

    const postsRes = await fetch(`/api/posts/${user.id}`);
    const posts = await postsRes.json();

    console.log(posts);
  } catch (error) {
    console.error(error);
  }
}
```

**Key differences:**

| Promise               | async/await     |
| --------------------- | --------------- |
| Returns Promise       | Returns Promise |
| `.then()` chains      | Sequential code |
| `.catch()` for errors | try/catch       |
| Can be verbose        | More readable   |

**Parallel execution:**

```javascript
// ❌ SLOW: Sequential (6 seconds total)
const user = await fetch("/api/user"); // 3s
const posts = await fetch("/api/posts"); // 3s

// ✅ FAST: Parallel (3 seconds total)
const [user, posts] = await Promise.all([
  fetch("/api/user"), // 3s
  fetch("/api/posts"), // 3s (parallel)
]);
```

**Error handling:**

```javascript
// Promise
fetch("/api/data")
  .then((res) => res.json())
  .catch((error) => {
    // Catches network errors AND JSON parsing errors
    console.error(error);
  });

// async/await
try {
  const res = await fetch("/api/data");
  const data = await res.json();
} catch (error) {
  // Catches network errors AND JSON parsing errors
  console.error(error);
}
```

**When to use which:**

- **async/await**: Sequential operations, cleaner syntax
- **Promise.all**: Parallel operations
- **Promise.race**: First to complete wins

---

### Q7: How do you handle race conditions?

**Answer:**

**Race condition:** Multiple async operations competing, last one "wins" incorrectly.

**Problem:**

```javascript
// ❌ RACE CONDITION
let searchId = 0;

async function search(query) {
  const currentId = ++searchId;
  const results = await fetchResults(query);

  // User typed "react" then quickly "redux"
  // "react" results arrive AFTER "redux" results
  // Shows "react" results for "redux" query!
  displayResults(results);
}
```

**Solution 1: Track latest request**

```javascript
// ✅ Ignore stale responses
let latestRequestId = 0;

async function search(query) {
  const requestId = ++latestRequestId;
  const results = await fetchResults(query);

  // Only process if still the latest request
  if (requestId === latestRequestId) {
    displayResults(results);
  }
}
```

**Solution 2: AbortController**

```javascript
// ✅ Cancel previous requests
let controller = new AbortController();

async function search(query) {
  // Cancel previous request
  controller.abort();
  controller = new AbortController();

  try {
    const results = await fetchResults(query, {
      signal: controller.signal,
    });
    displayResults(results);
  } catch (error) {
    if (error.name === "AbortError") {
      console.log("Request cancelled");
    }
  }
}
```

**Solution 3: Debounce**

```javascript
// ✅ Wait for user to stop typing
function debounce(fn, delay) {
  let timeoutId;

  return function (...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
}

const debouncedSearch = debounce(search, 300);

input.addEventListener("input", (e) => {
  debouncedSearch(e.target.value);
});
```

---

## 🎯 Part 3: Advanced Concepts

### Q8: What's the difference between `call`, `apply`, and `bind`?

**Answer:**

All three change `this` context, but differ in usage:

**`call(thisArg, ...args)`:**

```javascript
function greet(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

const person = { name: "Alice" };

greet.call(person, "Hello", "!"); // 'Hello, Alice!'
// Invokes immediately with arguments individually
```

**`apply(thisArg, [args])`:**

```javascript
greet.apply(person, ["Hello", "!"]); // 'Hello, Alice!'
// Invokes immediately with arguments as array
```

**`bind(thisArg, ...args)`:**

```javascript
const boundGreet = greet.bind(person, "Hello");
boundGreet("!"); // 'Hello, Alice!'
// Returns new function with bound this (doesn't invoke)
```

**Use cases:**

```javascript
// call/apply: Borrow methods
const numbers = [5, 6, 2, 3, 7];
const max = Math.max.apply(null, numbers); // 7

// Modern alternative: spread
const max = Math.max(...numbers);

// bind: Event handlers
class Component {
  constructor() {
    this.count = 0;
    // ✅ Bind this
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    this.count++; // this = component instance
  }
}

// Or use arrow function:
class Component {
  count = 0;

  handleClick = () => {
    this.count++; // Lexical this
  };
}
```

**Memory consideration:**

```javascript
// ❌ Creates new function every render
<button onClick={this.handleClick.bind(this)}>Click</button>

// ✅ Bind once in constructor
constructor() {
  this.handleClick = this.handleClick.bind(this);
}

<button onClick={this.handleClick}>Click</button>

// ✅ Or use arrow function (class field)
handleClick = () => { /* ... */ }

<button onClick={this.handleClick}>Click</button>
```

---

### Q9: Explain JavaScript's garbage collection

**Answer:**

JavaScript uses **automatic garbage collection** to free memory from unreachable objects.

**Mark-and-Sweep algorithm:**

```
1. Start from "roots" (global variables, call stack)
2. Mark all reachable objects
3. Sweep (delete) unmarked objects
```

**Reachability:**

```javascript
// ✅ Reachable (referenced)
let obj = { data: "Important" };

// ❌ Unreachable (no references)
obj = null; // Original object can be garbage collected
```

**Memory leaks - Common causes:**

**1. Global variables:**

```javascript
// ❌ LEAK: Accidentally global
function leak() {
  leakedVariable = "Oops"; // No var/let/const!
}

// ✅ FIX: Use strict mode
("use strict");
function noLeak() {
  leakedVariable = "Error!"; // ReferenceError
}
```

**2. Forgotten timers:**

```javascript
// ❌ LEAK: Timer never cleared
const interval = setInterval(() => {
  console.log("Running...");
}, 1000);

// ✅ FIX: Clear when done
const interval = setInterval(() => {
  console.log("Running...");
}, 1000);

// Later:
clearInterval(interval);
```

**3. Closures:**

```javascript
// ❌ LEAK: Closure keeps large data
function createHandler() {
  const largeData = new Array(1000000).fill("data");

  return function () {
    // Closure keeps largeData in memory!
    console.log(largeData.length);
  };
}

// ✅ FIX: Don't reference large data if not needed
function createHandler() {
  const largeData = new Array(1000000).fill("data");
  const length = largeData.length; // Store only what you need

  return function () {
    console.log(length); // Only keeps number, not array
  };
}
```

**4. DOM references:**

```javascript
// ❌ LEAK: Removed DOM still in memory
const button = document.getElementById("myButton");
button.addEventListener("click", handler);

// Remove from DOM
button.remove();
// But still referenced by `button` variable!

// ✅ FIX: Clear reference
button.removeEventListener("click", handler);
button = null;
```

**WeakMap/WeakSet for memory efficiency:**

```javascript
// ❌ Regular Map prevents garbage collection
const map = new Map();
let obj = { data: "Important" };
map.set(obj, "metadata");

obj = null; // obj can't be collected (Map still references it)

// ✅ WeakMap allows garbage collection
const weakMap = new WeakMap();
let obj = { data: "Important" };
weakMap.set(obj, "metadata");

obj = null; // obj CAN be collected (WeakMap doesn't prevent it)
```

---

### Q10: What are JavaScript modules (ESM vs CommonJS)?

**Answer:**

**ESM (ES Modules):**

```javascript
// math.js
export function add(a, b) {
  return a + b;
}

export const PI = 3.14159;

// main.js
import { add, PI } from "./math.js";

console.log(add(2, 3)); // 5
console.log(PI); // 3.14159
```

**CommonJS (Node.js):**

```javascript
// math.js
function add(a, b) {
  return a + b;
}

module.exports = { add };

// main.js
const { add } = require("./math.js");

console.log(add(2, 3)); // 5
```

**Key differences:**

| ESM                    | CommonJS                   |
| ---------------------- | -------------------------- |
| `import`/`export`      | `require`/`module.exports` |
| Static (compile-time)  | Dynamic (runtime)          |
| Async loading          | Sync loading               |
| Tree-shakeable         | Not tree-shakeable         |
| Strict mode by default | Not strict                 |
| Browser & Node.js      | Node.js only               |

**Tree shaking (dead code elimination):**

```javascript
// utils.js
export function used() {
  /* ... */
}
export function unused() {
  /* ... */
} // Not imported anywhere

// main.js
import { used } from "./utils.js";

// ✅ ESM: 'unused' removed from bundle (tree-shaken)
// ❌ CommonJS: 'unused' included in bundle
```

**Dynamic imports (ESM):**

```javascript
// Lazy load module
const button = document.getElementById("load");

button.addEventListener("click", async () => {
  const { renderChart } = await import("./chart.js");
  renderChart();
});
```

**Production rule: Use ESM (modern standard), CommonJS legacy only.**

---

## 🎯 Summary

**Core concepts to master:**

- ✅ Closures & scope
- ✅ Event loop (microtasks vs tasks)
- ✅ Prototypal inheritance
- ✅ `this` binding rules
- ✅ Promise vs async/await
- ✅ Race conditions & debouncing
- ✅ Garbage collection & memory leaks
- ✅ ESM vs CommonJS

**Interview tips:**

- Explain with examples
- Discuss trade-offs
- Mention edge cases
- Show production experience

Practice these until you can explain them while coding! 💪
