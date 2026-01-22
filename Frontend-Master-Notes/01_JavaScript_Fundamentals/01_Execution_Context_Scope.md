# 🎯 Execution Context & Scope

> Understanding how JavaScript executes code is fundamental to mastering the language. Every bug, performance issue, and design pattern relates back to these core concepts.

---

## 📖 1. Concept Explanation

### What is an Execution Context?

An **Execution Context** is an abstract concept that represents the environment in which JavaScript code is evaluated and executed. Think of it as a "box" that contains everything needed to run code: variables, functions, the value of `this`, and references to outer scopes.

### Types of Execution Contexts

1. **Global Execution Context (GEC)**
   - Created when JS file first runs
   - Only ONE per program
   - Creates `window` object (browser) or `global` (Node.js)
   - Sets `this` to the global object

2. **Function Execution Context (FEC)**
   - Created when a function is INVOKED (not defined)
   - New context for EACH function call
   - Has access to arguments object
   - Own `this` binding

3. **Eval Execution Context**
   - Created inside `eval()` function
   - Avoid in production (security + performance issues)

### Execution Context Lifecycle

```
┌─────────────────────────────────────┐
│  1. CREATION PHASE                  │
│     ├─ Create scope chain           │
│     ├─ Create variable object       │
│     ├─ Determine 'this' value       │
│     └─ Hoisting happens here        │
├─────────────────────────────────────┤
│  2. EXECUTION PHASE                 │
│     ├─ Assign values to variables   │
│     ├─ Execute function code        │
│     └─ Reference outer scopes       │
└─────────────────────────────────────┘
```

### Scope Chain

The scope chain is how JavaScript resolves variable names:

```
Current Scope → Outer Scope → ... → Global Scope → ReferenceError
```

---

## 🧠 2. Why It Matters in Real Projects

### Production Impact

**Memory Management:**

- Each execution context consumes memory
- Deep recursion can cause stack overflow
- Closures keep outer contexts alive (memory leaks)

**Performance:**

- Scope chain lookup has performance cost
- Deeply nested scopes = slower variable resolution
- V8 optimizes "hot" code paths differently

**Debugging:**

- Understanding call stack is critical
- Scope-related bugs are the hardest to debug
- `this` binding issues cause 80% of JS bugs

### Real-World Example

```javascript
// E-commerce cart - Common scope mistake
class ShoppingCart {
  constructor() {
    this.items = [];
  }

  addItem(product) {
    setTimeout(function () {
      // ❌ BUG: 'this' is undefined or window
      this.items.push(product);
    }, 100);
  }
}

// ✅ FIXED: Arrow function preserves 'this'
class ShoppingCart {
  constructor() {
    this.items = [];
  }

  addItem(product) {
    setTimeout(() => {
      this.items.push(product); // ✅ Correct 'this'
    }, 100);
  }
}
```

---

## ⚙️ 3. Internal Working

### How V8 Creates Execution Contexts

```javascript
function processOrder(orderId) {
  const tax = 0.08;

  function calculateTotal(price) {
    return price + price * tax;
  }

  return calculateTotal(100);
}

processOrder(42);
```

**Step-by-step execution:**

```
1. Global Execution Context Created
   ├─ window object created
   ├─ this = window
   └─ processOrder definition stored in memory

2. processOrder(42) called → New FEC created
   Creation Phase:
   ├─ Scope chain: [processOrder scope] → [Global scope]
   ├─ Variable object: { orderId: 42, tax: undefined, calculateTotal: fn }
   ├─ this = undefined (strict mode) or window (non-strict)
   └─ Hoisting: calculateTotal declared

   Execution Phase:
   ├─ tax = 0.08
   └─ calculateTotal(100) called → Another FEC created

3. calculateTotal(100) FEC created
   Creation Phase:
   ├─ Scope chain: [calculateTotal] → [processOrder] → [Global]
   ├─ Variable object: { price: 100 }
   └─ this = undefined (strict mode)

   Execution Phase:
   ├─ Access 'price' (found in current scope)
   ├─ Access 'tax' (NOT found, look in outer scope → processOrder)
   └─ Return 108

4. Stack unwinding
   ├─ calculateTotal FEC destroyed
   ├─ processOrder FEC destroyed
   └─ Back to Global EC
```

### Call Stack Visualization

```
┌─────────────────────────┐
│  calculateTotal(100)    │ ← Top of stack
├─────────────────────────┤
│  processOrder(42)       │
├─────────────────────────┤
│  Global Context         │ ← Bottom
└─────────────────────────┘
```

### Lexical Environment vs Variable Environment

**Before ES6:**

- Variable Object (VO) - holds vars, function declarations, arguments

**ES6+ (Modern):**

- **Lexical Environment** - holds let, const, function declarations
- **Variable Environment** - holds var declarations (legacy)

```javascript
function example() {
  var x = 1; // Variable Environment
  let y = 2; // Lexical Environment
  const z = 3; // Lexical Environment

  // Inner function accesses outer Lexical Environment
  function inner() {
    console.log(x, y, z); // Closure over all three
  }
  return inner;
}
```

---

## ✅ 4. Best Practices

### DO ✅

```javascript
// 1. Use arrow functions for lexical 'this'
class EventHandler {
  constructor() {
    this.count = 0;
  }

  attachListener() {
    button.addEventListener("click", () => {
      this.count++; // ✅ Correct 'this' binding
    });
  }
}

// 2. Minimize scope chain depth
function optimized() {
  const config = getConfig(); // Cache in local scope
  for (let i = 0; i < 1000; i++) {
    processItem(config); // Fast local access
  }
}

// 3. Use block scope to limit variable lifetime
{
  const tempData = fetchData();
  processData(tempData);
  // tempData eligible for GC after block
}

// 4. Explicit 'this' binding when needed
const boundMethod = obj.method.bind(obj);
```

### DON'T ❌

```javascript
// 1. Don't pollute global scope
// ❌ BAD
var userData = {};
var config = {};

// ✅ GOOD
const APP = {
  userData: {},
  config: {},
};

// 2. Don't rely on hoisting
// ❌ BAD
console.log(x); // undefined (confusing)
var x = 5;

// ✅ GOOD
const x = 5;
console.log(x); // Clear intent

// 3. Don't create accidental globals
// ❌ BAD
function calculate() {
  result = 42; // Implicit global!
}

// ✅ GOOD
function calculate() {
  "use strict";
  const result = 42;
  return result;
}
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Loop + Async + var

```javascript
// ❌ CLASSIC BUG
for (var i = 0; i < 5; i++) {
  setTimeout(function () {
    console.log(i); // Logs 5, 5, 5, 5, 5
  }, i * 100);
}

// WHY? All callbacks share same 'i' from function scope
// When callbacks execute, loop finished, i = 5

// ✅ FIX 1: Use let (block scope)
for (let i = 0; i < 5; i++) {
  setTimeout(function () {
    console.log(i); // Logs 0, 1, 2, 3, 4
  }, i * 100);
}

// ✅ FIX 2: IIFE (old way)
for (var i = 0; i < 5; i++) {
  (function (j) {
    setTimeout(function () {
      console.log(j); // Logs 0, 1, 2, 3, 4
    }, j * 100);
  })(i);
}
```

### Mistake #2: Lost 'this' Context

```javascript
// ❌ COMMON IN REACT CLASS COMPONENTS
class DataFetcher extends React.Component {
  constructor() {
    super();
    this.state = { data: null };
  }

  fetchData() {
    fetch('/api/data')
      .then(function(response) {
        // ❌ 'this' is undefined here
        this.setState({ data: response });
      });
  }
}

// ✅ FIX: Arrow function or bind
fetchData() {
  fetch('/api/data')
    .then(response => {
      this.setState({ data: response }); // ✅
    });
}
```

### Mistake #3: Unintentional Closures (Memory Leaks)

```javascript
// ❌ MEMORY LEAK IN SPA
function attachHandlers() {
  const hugeData = new Array(10000).fill("data");

  button.addEventListener("click", function () {
    console.log("clicked");
    // Closure keeps hugeData in memory even though unused
  });
}

// ✅ FIX: Don't capture unnecessary variables
function attachHandlers() {
  button.addEventListener("click", function () {
    const hugeData = new Array(10000).fill("data");
    console.log("clicked");
    // hugeData can be GC'd after function execution
  });
}
```

---

## 🔐 6. Security Considerations

### Avoiding Global Scope Pollution

```javascript
// ❌ VULNERABLE: Third-party scripts can modify
window.API_KEY = "secret123";

// ✅ SAFER: Use module scope
// config.js
const API_KEY = "secret123";
export function getApiKey() {
  return API_KEY;
}
```

### eval() Security Risks

```javascript
// ❌ NEVER DO THIS
const userInput = getUserInput();
eval(userInput); // User can execute arbitrary code!

// ✅ SAFE ALTERNATIVES
const result = JSON.parse(userInput); // For data
const fn = new Function("return " + userInput); // Still risky
```

### Strict Mode for Safety

```javascript
"use strict";

// Prevents accidental globals
function calculate() {
  result = 42; // ReferenceError (good!)
}

// Prevents deleting variables
var x = 5;
delete x; // SyntaxError (good!)
```

---

## 🚀 7. Performance Optimization

### 1. Reduce Scope Chain Lookups

```javascript
// ❌ SLOW: Repeated scope chain traversal
function processUsers(users) {
  for (let i = 0; i < users.length; i++) {
    if (users[i].age > window.APP_CONFIG.MIN_AGE) {
      // Slow
      users[i].eligible = true;
    }
  }
}

// ✅ FAST: Cache in local scope
function processUsers(users) {
  const minAge = window.APP_CONFIG.MIN_AGE; // One lookup
  for (let i = 0; i < users.length; i++) {
    if (users[i].age > minAge) {
      // Fast local access
      users[i].eligible = true;
    }
  }
}
```

### 2. Avoid Deep Nesting

```javascript
// ❌ SLOWER
function level1() {
  function level2() {
    function level3() {
      function level4() {
        return globalVar; // Long scope chain
      }
    }
  }
}

// ✅ FASTER: Flatten structure
function process(globalVar) {
  return globalVar; // Short scope chain
}
```

### 3. Use Block Scope for Garbage Collection

```javascript
// ❌ Keeps data in memory for entire function
function processLargeData() {
  const hugeArray = new Array(1000000).fill(0);
  const result = hugeArray.map((x) => x + 1);

  // More code...
  // hugeArray still in memory

  return result;
}

// ✅ Block scope allows earlier GC
function processLargeData() {
  const result = (() => {
    const hugeArray = new Array(1000000).fill(0);
    return hugeArray.map((x) => x + 1);
  })();
  // hugeArray eligible for GC now

  // More code...
  return result;
}
```

---

## 🧪 8. Code Examples

### Example 1: Scope Chain Resolution

```javascript
const global = "global";

function outer() {
  const outerVar = "outer";

  function inner() {
    const innerVar = "inner";

    console.log(innerVar); // 1. Found in inner scope
    console.log(outerVar); // 2. Found in outer scope
    console.log(global); // 3. Found in global scope
    console.log(notExists); // 4. ReferenceError
  }

  inner();
}

outer();

// Scope chain: inner → outer → global
```

### Example 2: 'this' Binding Rules

```javascript
// Rule 1: Default binding
function showThis() {
  console.log(this); // window (non-strict) or undefined (strict)
}

// Rule 2: Implicit binding
const obj = {
  name: "Object",
  showThis() {
    console.log(this.name); // 'Object'
  },
};
obj.showThis();

// Rule 3: Explicit binding
function greet() {
  console.log(`Hello, ${this.name}`);
}
const person = { name: "Alice" };
greet.call(person); // 'Hello, Alice'
greet.apply(person); // 'Hello, Alice'
const boundGreet = greet.bind(person);
boundGreet(); // 'Hello, Alice'

// Rule 4: new binding
function User(name) {
  this.name = name;
}
const user = new User("Bob"); // 'this' = new object

// Rule 5: Arrow function (lexical 'this')
const obj2 = {
  name: "Obj2",
  regularFunc: function () {
    setTimeout(function () {
      console.log(this.name); // undefined (lost context)
    }, 100);
  },
  arrowFunc: function () {
    setTimeout(() => {
      console.log(this.name); // 'Obj2' (lexical binding)
    }, 100);
  },
};
```

### Example 3: Closure and Context

```javascript
function createCounter() {
  let count = 0; // Private variable

  return {
    increment() {
      count++;
      return count;
    },
    decrement() {
      count--;
      return count;
    },
    getCount() {
      return count;
    },
  };
}

const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
console.log(counter.getCount()); // 2
console.log(counter.count); // undefined (private)

// Execution context of createCounter stays alive
// because returned object maintains closure
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: React Event Handlers

```javascript
// Problem: Lost 'this' in class components
class SearchBar extends React.Component {
  constructor(props) {
    super(props);
    this.state = { query: "" };

    // Solution 1: Bind in constructor
    this.handleSearch = this.handleSearch.bind(this);
  }

  handleSearch(e) {
    this.setState({ query: e.target.value });
  }

  // Solution 2: Class field with arrow function
  handleChange = (e) => {
    this.setState({ query: e.target.value });
  };

  render() {
    return (
      <input
        onChange={this.handleSearch} // Bound in constructor
        onBlur={this.handleChange} // Arrow function
      />
    );
  }
}
```

### Use Case 2: Module Pattern (Pre-ES6)

```javascript
const UserModule = (function () {
  // Private variables (closure)
  const users = [];
  let currentUser = null;

  // Private function
  function validateUser(user) {
    return user && user.name && user.email;
  }

  // Public API
  return {
    addUser(user) {
      if (validateUser(user)) {
        users.push(user);
        return true;
      }
      return false;
    },

    setCurrentUser(user) {
      if (users.includes(user)) {
        currentUser = user;
      }
    },

    getCurrentUser() {
      return currentUser;
    },
  };
})();

// Usage
UserModule.addUser({ name: "Alice", email: "alice@example.com" });
// users array is not accessible from outside
```

### Use Case 3: Debounce with Closure

```javascript
function debounce(func, delay) {
  let timeoutId; // Closure variable

  return function (...args) {
    const context = this; // Preserve 'this'

    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => {
      func.apply(context, args);
    }, delay);
  };
}

// Usage in search component
const SearchComponent = {
  query: "",

  search() {
    console.log(`Searching for: ${this.query}`);
    // API call here
  },

  handleInput: debounce(function (e) {
    this.query = e.target.value;
    this.search(); // 'this' correctly bound
  }, 300),
};
```

---

## ❓ 10. Interview Questions

### Q1: Explain the difference between scope and context in JavaScript.

**Answer:**

**Scope** refers to the visibility/accessibility of variables. It's about WHERE you can access a variable (lexical environment).

**Context** refers to the value of `this` within a function. It's about HOW a function is called (execution context).

```javascript
const obj = {
  name: "MyObject",
  showScope() {
    const localVar = "scoped"; // Scope: function scope
    console.log(this.name); // Context: 'this' refers to obj
  },
};
```

**Follow-up:** How does 'this' work in arrow functions?

- Arrow functions don't have their own `this`. They inherit `this` from the enclosing lexical scope (where they're defined, not called).

---

### Q2: What happens in the creation phase vs execution phase of an execution context?

**Answer:**

**Creation Phase:**

1. Create scope chain
2. Create Variable Object (or Lexical Environment)
   - Hoist function declarations (fully)
   - Hoist variable declarations (initialize as undefined for `var`, uninitialized for `let`/`const`)
3. Determine `this` value

**Execution Phase:**

1. Assign values to variables
2. Execute code line by line
3. Handle function invocations (create new contexts)

```javascript
console.log(x); // undefined (hoisted, not assigned)
var x = 5;
console.log(x); // 5 (assigned in execution phase)
```

---

### Q3: How do closures relate to execution contexts?

**Answer:**

A closure is created when a function "remembers" variables from its outer scope, even after the outer function has returned. This happens because:

1. When the outer function executes, it creates an execution context
2. The inner function maintains a reference to the outer context's Lexical Environment
3. Even when the outer context is popped off the call stack, the Lexical Environment stays in memory
4. This is how the inner function can still access outer variables

```javascript
function outer() {
  const secret = "data"; // In outer's Lexical Environment

  return function inner() {
    console.log(secret); // Closure: access outer's Lexical Environment
  };
}

const closure = outer();
// outer() execution context destroyed
// But its Lexical Environment kept alive by closure

closure(); // 'data' - still accessible
```

**Memory implication:** Closures can cause memory leaks if not managed properly.

---

### Q4: Explain the Temporal Dead Zone (TDZ).

**Answer:**

The TDZ is the time between entering a scope and the actual declaration/initialization of a `let` or `const` variable. During this time, accessing the variable throws a ReferenceError.

```javascript
console.log(x); // ReferenceError: Cannot access 'x' before initialization
let x = 5;

// TDZ starts at scope entry
{
  // TDZ for y starts here
  console.log(y); // ReferenceError
  let y = 10; // TDZ ends here
}
```

**Why it exists:** To catch errors. `var` hoisting with `undefined` hides bugs. TDZ makes temporal errors explicit.

---

### Q5: How does JavaScript handle recursive function calls and what causes stack overflow?

**Answer:**

Each recursive call creates a new execution context pushed onto the call stack. The stack has a fixed size (typically ~10k-100k frames depending on engine and environment).

**Stack Overflow:**

```javascript
function recursiveWithoutBase() {
  recursiveWithoutBase(); // No base case
}
// RangeError: Maximum call stack size exceeded
```

**Tail Call Optimization (TCO):**
ES6 spec includes TCO for tail-recursive functions in strict mode, but most engines don't fully implement it.

```javascript
// NOT tail-recursive (needs to return to do math)
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1); // Not in tail position
}

// Tail-recursive (last operation is recursive call)
function factorialTCO(n, acc = 1) {
  if (n <= 1) return acc;
  return factorialTCO(n - 1, n * acc); // Tail position
}
```

**Production solution:** Use iteration or trampoline functions for deep recursion.

---

## 🧩 11. Design Patterns

### Pattern 1: Module Pattern (Closure-based)

**Pros:**

- Encapsulation (private variables)
- Clean public API
- No global pollution

**Cons:**

- Memory overhead (each instance gets own methods)
- No inheritance support
- Harder to test private functions

```javascript
const Calculator = (function () {
  let history = []; // Private

  return {
    add(a, b) {
      const result = a + b;
      history.push(result);
      return result;
    },
    getHistory() {
      return [...history]; // Return copy
    },
  };
})();
```

**When to use:** Small utilities, libraries (pre-ES6 modules)

---

### Pattern 2: Revealing Module Pattern

```javascript
const UserManager = (function () {
  const users = [];

  function addUser(user) {
    users.push(user);
  }

  function getUsers() {
    return users;
  }

  // Reveal only what's needed
  return {
    add: addUser,
    getAll: getUsers,
  };
})();
```

**Pros:** Clear separation of public/private, easier to refactor

---

### Pattern 3: IIFE for Scope Isolation

```javascript
// Isolate variables from global scope
(function () {
  const config = { apiKey: "secret" };
  // Use config
})();
// config not accessible here

// Useful in older codebases and inline scripts
```

---

## 🎯 Summary

Execution contexts and scope are the foundation of JavaScript execution. Master these concepts to:

- Debug effectively (understand call stack)
- Write performant code (optimize scope chain)
- Avoid memory leaks (manage closures)
- Design clean APIs (leverage scope for encapsulation)
- Ace interviews (most JS questions trace back to these fundamentals)

**Key Takeaway:** JavaScript is NOT compiled in the traditional sense, but it's NOT purely interpreted either. The engine creates execution contexts during a "compilation-like" phase (hoisting, scope determination) before executing code. Understanding this two-phase process is critical for senior engineers.
