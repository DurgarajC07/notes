# 🔗 Hoisting & Temporal Dead Zone (TDZ)

> Hoisting is one of JavaScript's most misunderstood features. It's not about "moving code up"—it's about how the engine creates execution contexts.

---

## 📖 1. Concept Explanation

### What is Hoisting?

**Hoisting** is JavaScript's behavior of processing declarations during the **creation phase** of execution context, before code executes. This makes it appear as if declarations are "moved to the top" of their scope.

**Reality:** Nothing moves. The engine:

1. First pass: Scan for declarations (creation phase)
2. Second pass: Execute code line-by-line (execution phase)

### What Gets Hoisted?

| Declaration Type         | Hoisted?                 | Initialized?     | Accessible Before Declaration? |
| ------------------------ | ------------------------ | ---------------- | ------------------------------ |
| `var`                    | ✅ Yes                   | ✅ `undefined`   | ✅ Returns `undefined`         |
| `let`                    | ✅ Yes                   | ❌ No (TDZ)      | ❌ ReferenceError              |
| `const`                  | ✅ Yes                   | ❌ No (TDZ)      | ❌ ReferenceError              |
| `function` (declaration) | ✅ Yes                   | ✅ Full function | ✅ Fully usable                |
| `function` (expression)  | Depends on var/let/const | No               | ❌ TypeError/ReferenceError    |
| `class`                  | ✅ Yes                   | ❌ No (TDZ)      | ❌ ReferenceError              |

### Temporal Dead Zone (TDZ)

The **TDZ** is the time period between:

- **Start:** Entering the scope where a `let`/`const`/`class` is declared
- **End:** The line where it's actually declared/initialized

```javascript
// TDZ starts at scope entry
{
  console.log(x); // ❌ ReferenceError (in TDZ)
  let x = 5; // TDZ ends here
}
```

---

## 🧠 2. Why It Matters in Real Projects

### Production Impact

**1. Hidden Bugs:**

```javascript
// ❌ Looks like 'product' is defined, but it's not
if (condition) {
  console.log(product); // undefined (not ReferenceError!)
  var product = getProduct(); // Hoisted to function scope
}
```

**2. Order Dependencies:**

```javascript
// ❌ Works with function declarations
greet(); // "Hello"
function greet() {
  console.log("Hello");
}

// ❌ Fails with function expressions
sayHi(); // TypeError: sayHi is not a function
var sayHi = function () {
  console.log("Hi");
};
```

**3. Performance:**

```javascript
// ❌ Bad: Creates new function in memory on every render
function Component() {
  var handler = function () {
    /* ... */
  }; // Recreated each render
  return <button onClick={handler} />;
}

// ✅ Good: Hoist function declaration (if pure)
function Component() {
  function handler() {
    /* ... */
  } // Same function reference
  return <button onClick={handler} />;
}
```

**4. Interview Red Flags:**

- Not understanding hoisting suggests shallow JS knowledge
- Using `var` in 2026 shows lack of modern JS awareness
- Not knowing TDZ means can't debug scope errors

---

## ⚙️ 3. Internal Working

### How V8 Processes Declarations

```javascript
console.log(a); // undefined
var a = 5;
console.log(a); // 5
```

**What V8 Actually Does:**

**Creation Phase:**

```javascript
// Variable Object (VO) created
{
  a: undefined; // var hoisted and initialized
}
```

**Execution Phase:**

```javascript
console.log(a); // Read from VO: undefined
a = 5; // Update VO
console.log(a); // Read from VO: 5
```

### Function Hoisting (Full)

```javascript
greet(); // "Hello" - works!

function greet() {
  console.log("Hello");
}
```

**Creation Phase:**

```javascript
{
  greet: <function object> // Fully available
}
```

### let/const Hoisting (Partial - TDZ)

```javascript
console.log(x); // ReferenceError
let x = 5;
```

**Creation Phase:**

```javascript
{
  x: <uninitialized> // Exists but not accessible (TDZ)
}
```

**Execution Phase:**

```javascript
// Line 1: x exists but throws ReferenceError if accessed
// Line 2: x = 5, TDZ ends, now accessible
```

### Class Hoisting

```javascript
const instance = new MyClass(); // ReferenceError

class MyClass {
  constructor() {
    this.name = "Example";
  }
}
```

**Why?** Classes have TDZ just like `let`/`const` to:

1. Prevent usage before definition
2. Catch errors early
3. Support super() calls correctly

---

## ✅ 4. Best Practices

### DO ✅

```javascript
// 1. Declare variables at the top of scope
function processUser(user) {
  const name = user.name; // ✅ Clear intent
  const age = user.age;
  let status = "processing";

  // Logic here
}

// 2. Use const by default, let when needed
const API_URL = "https://api.example.com"; // ✅ Won't change
let counter = 0; // ✅ Will change

// 3. Use function declarations for top-level functions
function calculateTax(amount) {
  // ✅ Hoisted, clear
  return amount * 0.08;
}

// 4. Use function expressions for conditional logic
let getDiscount;
if (isPremiumUser) {
  getDiscount = function () {
    return 0.2;
  }; // ✅ Conditional
} else {
  getDiscount = function () {
    return 0.1;
  };
}

// 5. Declare all variables before loops
const items = getItems(); // ✅
const threshold = getThreshold(); // ✅
for (let i = 0; i < items.length; i++) {
  if (items[i] > threshold) {
    // ...
  }
}
```

### DON'T ❌

```javascript
// 1. Don't use var (use let/const)
var x = 5; // ❌ Function-scoped, hoisted to undefined

// 2. Don't rely on hoisting for logic
greet(); // ❌ Confusing to readers
function greet() {
  console.log("Hi");
}

// 3. Don't access variables before declaration
console.log(x); // ❌ Will fail or be undefined
let x = 5;

// 4. Don't redeclare variables
var user = "Alice";
var user = "Bob"; // ❌ Allowed but error-prone

// 5. Don't use function expressions when declaration works
const add = function (a, b) {
  // ❌ Unnecessarily complex
  return a + b;
};
// ✅ Better:
function add(a, b) {
  return a + b;
}
```

---

## ❌ 5. Common Mistakes

### Mistake #1: var in Loops

```javascript
// ❌ CLASSIC BUG
for (var i = 0; i < 3; i++) {
  setTimeout(function () {
    console.log(i); // 3, 3, 3 (all reference same 'i')
  }, i * 100);
}

// Why? 'var' is function-scoped, not block-scoped
// All closures reference the SAME 'i'

// ✅ FIX: Use let (block-scoped)
for (let i = 0; i < 3; i++) {
  setTimeout(function () {
    console.log(i); // 0, 1, 2 (each has own 'i')
  }, i * 100);
}
```

### Mistake #2: Function Expression Before Declaration

```javascript
// ❌ TypeError: fetchData is not a function
const result = fetchData();

const fetchData = function () {
  return { data: "example" };
};

// Why? 'fetchData' hoisted as undefined (const with TDZ)
// Accessing before initialization throws error

// ✅ FIX: Declare before use
const fetchData = function () {
  return { data: "example" };
};
const result = fetchData();
```

### Mistake #3: Conditional Function Declarations

```javascript
// ❌ UNPREDICTABLE BEHAVIOR
if (condition) {
  function doSomething() {
    console.log("Version 1");
  }
} else {
  function doSomething() {
    console.log("Version 2");
  }
}

// Problem: Function declarations hoisted out of blocks
// Behavior differs between browsers/modes

// ✅ FIX: Use function expressions
let doSomething;
if (condition) {
  doSomething = function () {
    console.log("Version 1");
  };
} else {
  doSomething = function () {
    console.log("Version 2");
  };
}
```

### Mistake #4: Accessing Class Before Declaration

```javascript
// ❌ ReferenceError: Cannot access 'User' before initialization
const user = new User("Alice");

class User {
  constructor(name) {
    this.name = name;
  }
}

// ✅ FIX: Declare class first
class User {
  constructor(name) {
    this.name = name;
  }
}
const user = new User("Alice");
```

### Mistake #5: Redeclaring with var

```javascript
// ❌ No error, but confusing
var config = { theme: "dark" };

// 100 lines later...
var config = { theme: "light" }; // Silently overwrites!

// ✅ FIX: Use const/let (throws SyntaxError on redeclaration)
const config = { theme: "dark" };
const config = { theme: "light" }; // SyntaxError: Identifier 'config' has already been declared
```

---

## 🔐 6. Security Considerations

### Temporal Dead Zone Prevents Exploits

```javascript
// ❌ With var: Potential timing attack
if (Math.random() > 0.5) {
  var secret = getSecretKey(); // Hoisted, may leak as 'undefined'
}
console.log(secret); // undefined if condition false (info leak)

// ✅ With let: Throws error, no info leak
if (Math.random() > 0.5) {
  let secret = getSecretKey();
}
console.log(secret); // ReferenceError (secure)
```

### Global Scope Pollution

```javascript
// ❌ DANGEROUS: Implicit global
function processData() {
  data = [1, 2, 3]; // No var/let/const = global!
}
processData();
console.log(window.data); // [1, 2, 3] - global pollution

// Attacker can override
window.data = maliciousData;

// ✅ SAFE: Strict mode prevents
("use strict");
function processData() {
  data = [1, 2, 3]; // ReferenceError: data is not defined
}
```

### Function Hoisting in eval()

```javascript
// ❌ NEVER USE: eval() with hoisting
const userCode = `
  console.log(password); // undefined (hoisted)
  var password = 'secret123';
`;
eval(userCode); // Unpredictable, dangerous

// ✅ SAFE: Avoid eval() entirely
```

---

## 🚀 7. Performance Optimization

### 1. Function Declarations vs Expressions (Performance)

```javascript
// ✅ FASTER: Function declaration (hoisted, single allocation)
function add(a, b) {
  return a + b;
}

// ❌ SLOWER: Function expression (allocated on each execution)
for (let i = 0; i < 1000; i++) {
  const add = function (a, b) {
    return a + b;
  }; // 1000 allocations!
  add(i, 1);
}

// ✅ OPTIMIZED: Hoist out of loop
const add = function (a, b) {
  return a + b;
};
for (let i = 0; i < 1000; i++) {
  add(i, 1); // Single allocation
}
```

### 2. Avoid TDZ Checks in Hot Paths

```javascript
// ❌ SLOWER: TDZ check on every iteration
for (let i = 0; i < 10000; i++) {
  let result = compute(i); // TDZ check (minimal but exists)
  process(result);
}

// ✅ FASTER: Declare outside loop (V8 optimizes better)
let result;
for (let i = 0; i < 10000; i++) {
  result = compute(i);
  process(result);
}
```

### 3. const vs let Performance

```javascript
// const gives hints to optimizer
const BASE_URL = "https://api.example.com"; // V8 knows this won't change

// let requires tracking for reassignment
let counter = 0; // V8 must check for reassignments
```

**Note:** Performance difference is negligible in modern engines, but `const` enables optimizations like constant folding.

---

## 🧪 8. Code Examples

### Example 1: Hoisting Puzzle

```javascript
var x = 1;

function test() {
  console.log(x); // What will this log?
  var x = 2;
  console.log(x); // What will this log?
}

test();

// Output:
// undefined  (not 1! x is hoisted in function scope)
// 2

// Equivalent code after hoisting:
function test() {
  var x; // Hoisted to top, initialized as undefined
  console.log(x); // undefined
  x = 2;
  console.log(x); // 2
}
```

### Example 2: TDZ with typeof

```javascript
// typeof is safe for undeclared variables
console.log(typeof notDeclared); // "undefined" (no error)

// But NOT safe in TDZ
console.log(typeof x); // ReferenceError (in TDZ)
let x = 5;
```

### Example 3: Function Hoisting Priority

```javascript
console.log(foo); // What gets logged?

var foo = "variable";

function foo() {
  return "function";
}

console.log(foo); // What gets logged?

// Output:
// [Function: foo]  (function hoisted with value)
// 'variable'       (var assignment happens during execution)

// Hoisting priority: function declarations > var declarations
```

### Example 4: Block Scope with let/const

```javascript
{
  console.log(x); // ReferenceError (TDZ)
  let x = 5;
}

{
  let y = 10;
}
console.log(y); // ReferenceError (block-scoped)

// vs var (function/global scoped)
{
  var z = 15;
}
console.log(z); // 15 (leaked out of block)
```

### Example 5: Class Hoisting

```javascript
// ❌ Doesn't work
const instance = new Rectangle(10, 5);

class Rectangle {
  constructor(width, height) {
    this.width = width;
    this.height = height;
  }
}

// ✅ Works
class Circle {
  constructor(radius) {
    this.radius = radius;
  }
}
const circle = new Circle(10);
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: React Hooks (Hoisting Issues)

```javascript
// ❌ MISTAKE: Conditional hook (relies on hoisting misconception)
function UserProfile({ userId }) {
  if (!userId) {
    return <div>Loading...</div>;
  }

  // ❌ WRONG: Hooks must be at top level
  const [user, setUser] = useState(null); // TDZ if moved

  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);

  return <div>{user?.name}</div>;
}

// ✅ CORRECT: Hooks always at top
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    if (userId) {
      fetchUser(userId).then(setUser);
    }
  }, [userId]);

  if (!userId) {
    return <div>Loading...</div>;
  }

  return <div>{user?.name}</div>;
}
```

### Use Case 2: Module Pattern with Hoisting

```javascript
// ❌ BAD: Relies on hoisting
const UserService = (function () {
  return {
    getUser, // Function hoisted
    createUser,
  };

  function getUser(id) {
    return users.find((u) => u.id === id);
  }

  function createUser(data) {
    users.push(data);
  }

  var users = []; // Hoisted as undefined!
})();

// ✅ GOOD: Clear order
const UserService = (function () {
  const users = []; // Declared first

  function getUser(id) {
    return users.find((u) => u.id === id);
  }

  function createUser(data) {
    users.push(data);
  }

  return {
    getUser,
    createUser,
  };
})();
```

### Use Case 3: Event Handler Scope

```javascript
// ❌ COMMON MISTAKE
function setupHandlers() {
  for (var i = 0; i < buttons.length; i++) {
    buttons[i].onclick = function () {
      console.log("Button " + i + " clicked"); // Always logs last value
    };
  }
}

// ✅ FIX 1: Use let (block scope)
function setupHandlers() {
  for (let i = 0; i < buttons.length; i++) {
    buttons[i].onclick = function () {
      console.log("Button " + i + " clicked"); // Correct value
    };
  }
}

// ✅ FIX 2: IIFE (old way)
function setupHandlers() {
  for (var i = 0; i < buttons.length; i++) {
    (function (index) {
      buttons[index].onclick = function () {
        console.log("Button " + index + " clicked");
      };
    })(i);
  }
}

// ✅ FIX 3: Modern approach (dataset)
buttons.forEach((button, index) => {
  button.dataset.index = index;
  button.onclick = function () {
    console.log("Button " + this.dataset.index + " clicked");
  };
});
```

---

## ❓ 10. Interview Questions

### Q1: What is hoisting? Give an example.

**Answer:**

Hoisting is JavaScript's behavior of processing declarations during the creation phase of execution context, before code executes. It's not "moving code up"—it's the engine's two-phase execution.

```javascript
console.log(x); // undefined (not ReferenceError)
var x = 5;

greet(); // "Hello" (works!)
function greet() {
  console.log("Hello");
}
```

**Internally:**

1. **Creation phase:** `var x` hoisted (set to `undefined`), `function greet` fully hoisted
2. **Execution phase:** Line-by-line execution, `x` assigned 5

**Follow-up:** What about `let` and `const`?

- They ARE hoisted, but not initialized (Temporal Dead Zone)
- Accessing before declaration throws ReferenceError

---

### Q2: Explain the Temporal Dead Zone.

**Answer:**

The TDZ is the period from scope entry until a `let`/`const`/`class` declaration, during which accessing the variable throws a ReferenceError.

```javascript
{
  // TDZ starts
  console.log(x); // ReferenceError
  let x = 5; // TDZ ends
}
```

**Why it exists:**

1. Catch bugs early (prevent usage before initialization)
2. Support for `const` (must initialize at declaration)
3. Better optimization hints for engines

**Follow-up:** Can you check if a variable exists without throwing?

- `typeof` works for undeclared variables, NOT for TDZ variables

---

### Q3: What's the difference between function declarations and function expressions regarding hoisting?

**Answer:**

| Aspect                   | Function Declaration            | Function Expression                    |
| ------------------------ | ------------------------------- | -------------------------------------- |
| Hoisting                 | Fully hoisted (can call before) | Hoisted as variable (undefined or TDZ) |
| Usage before declaration | ✅ Works                        | ❌ TypeError or ReferenceError         |

```javascript
// Function Declaration
greet(); // Works!
function greet() {
  console.log("Hi");
}

// Function Expression (var)
sayHi(); // TypeError: sayHi is not a function
var sayHi = function () {
  console.log("Hi");
};

// Function Expression (let/const)
wave(); // ReferenceError (TDZ)
const wave = function () {
  console.log("Wave");
};
```

**When to use which:**

- **Declaration:** Top-level utilities, want hoisting
- **Expression:** Conditional logic, callbacks, no hoisting needed

---

### Q4: Why doesn't this code work as expected?

```javascript
for (var i = 0; i < 5; i++) {
  setTimeout(function () {
    console.log(i);
  }, i * 100);
}
// Logs: 5, 5, 5, 5, 5
```

**Answer:**

**Problem:** `var` is function-scoped (or global if not in function), not block-scoped. All closures capture the SAME `i` variable. By the time callbacks execute, the loop has finished and `i = 5`.

**Visualized:**

```javascript
var i; // Single 'i' for entire loop
for (i = 0; i < 5; i++) {
  setTimeout(function () {
    console.log(i); // All reference same 'i'
  }, i * 100);
}
// Now i = 5, callbacks execute
```

**Solutions:**

1. **Use `let` (block scope):**

```javascript
for (let i = 0; i < 5; i++) {
  // New 'i' for each iteration
  setTimeout(function () {
    console.log(i); // 0, 1, 2, 3, 4
  }, i * 100);
}
```

2. **IIFE (old way):**

```javascript
for (var i = 0; i < 5; i++) {
  (function (j) {
    // Create new scope with copy of 'i'
    setTimeout(function () {
      console.log(j);
    }, j * 100);
  })(i);
}
```

---

### Q5: What will this output and why?

```javascript
var a = 1;

function outer() {
  console.log(a);
  var a = 2;
  console.log(a);
}

outer();
```

**Answer:**

```
Output:
undefined
2
```

**Why:**

The `var a` inside `outer` is hoisted to the top of the function scope, shadowing the global `a`.

**Equivalent code:**

```javascript
var a = 1;

function outer() {
  var a; // Hoisted, initialized as undefined
  console.log(a); // undefined (local 'a', not global)
  a = 2;
  console.log(a); // 2
}
```

**Key insight:** Function-scoped variables shadow outer variables, even before assignment.

---

### Q6: Can you hoist an arrow function?

**Answer:**

No, arrow functions can't be hoisted in the way function declarations are. They're always function expressions.

```javascript
// ❌ Doesn't work
greet(); // ReferenceError or TypeError
const greet = () => console.log("Hi");

// ✅ Works (function declaration)
sayHi(); // "Hi"
function sayHi() {
  console.log("Hi");
}
```

**Why:** Arrow functions are syntactic sugar for function expressions. They follow the hoisting rules of the variable they're assigned to (`var`, `let`, or `const`).

---

## 🧩 11. Design Patterns

### Pattern 1: Immediately Invoked Function Expression (IIFE)

**Purpose:** Create private scope without polluting global namespace.

```javascript
// Module pattern with IIFE
const CounterModule = (function () {
  let count = 0; // Private variable

  // Private function
  function validateIncrement(value) {
    return value > 0;
  }

  // Public API
  return {
    increment(value = 1) {
      if (validateIncrement(value)) {
        count += value;
      }
      return count;
    },
    getCount() {
      return count;
    },
  };
})();

// Usage
CounterModule.increment(5); // 5
CounterModule.getCount(); // 5
// count is not accessible from outside
```

**Pros:**

- Private variables (before ES6 modules)
- No global pollution
- Runs immediately

**Cons:**

- Harder to debug (anonymous function in stack trace)
- Not reusable (single instance)
- ES6 modules are better for new code

---

### Pattern 2: Lazy Initialization (Hoisting-Aware)

```javascript
// ❌ BAD: Initializes on every call
function getConfig() {
  const config = expensiveConfigSetup(); // Runs every time
  return config;
}

// ✅ GOOD: Lazy initialization with closure
const getConfig = (function () {
  let config; // Hoisted, undefined initially

  return function () {
    if (!config) {
      config = expensiveConfigSetup(); // Runs once
    }
    return config;
  };
})();
```

**When to use:** Expensive initialization that might not be needed.

---

### Pattern 3: Revealing Module Pattern

```javascript
const UserService = (function () {
  // Private data
  const users = [];

  // Private functions
  function validateUser(user) {
    return user && user.name && user.email;
  }

  function findUserIndex(id) {
    return users.findIndex((u) => u.id === id);
  }

  // Public functions
  function addUser(user) {
    if (validateUser(user)) {
      users.push(user);
      return true;
    }
    return false;
  }

  function getUser(id) {
    return users.find((u) => u.id === id);
  }

  // Reveal only public API
  return {
    add: addUser,
    get: getUser,
  };
})();
```

**Pros:**

- Clear separation of public/private
- Easy to see public API at bottom
- Consistent naming (no need for different names)

---

## 🎯 Summary

Hoisting and TDZ are fundamental to understanding JavaScript's execution model:

- **Hoisting:** Declarations processed in creation phase (before execution)
- **var:** Hoisted and initialized to `undefined`
- **let/const:** Hoisted but uninitialized (TDZ)
- **function:** Fully hoisted (can call before declaration)
- **class:** Hoisted but in TDZ (like let/const)

**Best practices:**

1. Use `const` by default, `let` when reassignment needed, never `var`
2. Declare variables at the top of scope
3. Understand TDZ to debug ReferenceErrors
4. Prefer function declarations for top-level, expressions for callbacks

**Interview tip:** Hoisting questions test whether you understand JavaScript's two-phase execution model. Always explain the creation phase vs execution phase.
