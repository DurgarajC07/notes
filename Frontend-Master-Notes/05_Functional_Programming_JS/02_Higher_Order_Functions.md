# Higher-Order Functions

> **Functions that operate on other functions: map, filter, reduce, and more**

---

## 🎯 Overview

Higher-order functions (HOFs) are functions that take functions as arguments or return functions. They're fundamental to functional programming and enable powerful abstractions and code reuse.

---

## 📚 Core Concepts

### **Concept 1: What Are Higher-Order Functions?**

A higher-order function either:

1. **Takes one or more functions as arguments**
2. **Returns a function**

```javascript
// HOF: takes function as argument
function map(array, fn) {
  const result = [];
  for (const item of array) {
    result.push(fn(item));
  }
  return result;
}

const doubled = map([1, 2, 3], (x) => x * 2);
// [2, 4, 6]

// HOF: returns a function
function multiplier(factor) {
  return function (number) {
    return number * factor;
  };
}

const double = multiplier(2);
const triple = multiplier(3);

double(5); // 10
triple(5); // 15
```

### **Concept 2: Built-in Array HOFs**

**Array.prototype.map** - Transform each element:

```javascript
const numbers = [1, 2, 3, 4];

// Double each number
const doubled = numbers.map((n) => n * 2);
// [2, 4, 6, 8]

// Extract property
const users = [
  { id: 1, name: "Alice" },
  { id: 2, name: "Bob" },
];
const names = users.map((user) => user.name);
// ['Alice', 'Bob']

// Transform to objects
const items = [1, 2, 3];
const objects = items.map((id) => ({ id, value: id * 10 }));
// [{ id: 1, value: 10 }, { id: 2, value: 20 }, { id: 3, value: 30 }]
```

**Array.prototype.filter** - Select elements that pass a test:

```javascript
const numbers = [1, 2, 3, 4, 5, 6];

// Get even numbers
const evens = numbers.filter((n) => n % 2 === 0);
// [2, 4, 6]

// Filter objects
const users = [
  { name: "Alice", age: 25 },
  { name: "Bob", age: 17 },
  { name: "Carol", age: 30 },
];
const adults = users.filter((user) => user.age >= 18);
// [{ name: 'Alice', age: 25 }, { name: 'Carol', age: 30 }]

// Remove falsy values
const mixed = [0, 1, false, 2, "", 3, null, 4];
const truthy = mixed.filter(Boolean);
// [1, 2, 3, 4]
```

**Array.prototype.reduce** - Reduce array to single value:

```javascript
const numbers = [1, 2, 3, 4];

// Sum
const sum = numbers.reduce((acc, n) => acc + n, 0);
// 10

// Product
const product = numbers.reduce((acc, n) => acc * n, 1);
// 24

// Group by property
const users = [
  { name: "Alice", role: "admin" },
  { name: "Bob", role: "user" },
  { name: "Carol", role: "admin" },
];

const byRole = users.reduce((acc, user) => {
  if (!acc[user.role]) acc[user.role] = [];
  acc[user.role].push(user);
  return acc;
}, {});
// { admin: [Alice, Carol], user: [Bob] }

// Count occurrences
const fruits = ["apple", "banana", "apple", "orange", "banana", "apple"];
const counts = fruits.reduce((acc, fruit) => {
  acc[fruit] = (acc[fruit] || 0) + 1;
  return acc;
}, {});
// { apple: 3, banana: 2, orange: 1 }
```

### **Concept 3: Other Built-in HOFs**

```javascript
// find - first element that passes test
const users = [{ id: 1 }, { id: 2 }, { id: 3 }];
const user = users.find((u) => u.id === 2);
// { id: 2 }

// some - true if any element passes test
const hasEven = [1, 3, 5, 6].some((n) => n % 2 === 0);
// true

// every - true if all elements pass test
const allEven = [2, 4, 6].every((n) => n % 2 === 0);
// true

// sort - sort with comparator function
const nums = [3, 1, 4, 1, 5];
nums.sort((a, b) => a - b);
// [1, 1, 3, 4, 5]

const users = [{ age: 30 }, { age: 20 }, { age: 25 }];
users.sort((a, b) => a.age - b.age);
// [{ age: 20 }, { age: 25 }, { age: 30 }]
```

---

## 🔥 Practical Examples

### **Example 1: Chaining HOFs**

```javascript
const users = [
  { name: "Alice", age: 25, active: true },
  { name: "Bob", age: 17, active: false },
  { name: "Carol", age: 30, active: true },
  { name: "David", age: 22, active: true },
];

// Get names of active adult users
const activeAdultNames = users
  .filter((user) => user.active)
  .filter((user) => user.age >= 18)
  .map((user) => user.name)
  .sort();
// ['Alice', 'Carol', 'David']
```

### **Example 2: Custom HOFs**

```javascript
// Implement map
function map(array, fn) {
  const result = [];
  for (let i = 0; i < array.length; i++) {
    result.push(fn(array[i], i, array));
  }
  return result;
}

// Implement filter
function filter(array, predicate) {
  const result = [];
  for (let i = 0; i < array.length; i++) {
    if (predicate(array[i], i, array)) {
      result.push(array[i]);
    }
  }
  return result;
}

// Implement reduce
function reduce(array, reducer, initial) {
  let acc = initial;
  for (let i = 0; i < array.length; i++) {
    acc = reducer(acc, array[i], i, array);
  }
  return acc;
}
```

### **Example 3: Function Factories**

```javascript
// Create specialized functions
function greaterThan(n) {
  return function (x) {
    return x > n;
  };
}

const greaterThan10 = greaterThan(10);
[5, 15, 8, 20].filter(greaterThan10); // [15, 20]

// Property accessor
function pluck(property) {
  return function (obj) {
    return obj[property];
  };
}

const users = [{ name: "Alice" }, { name: "Bob" }];
const names = users.map(pluck("name")); // ['Alice', 'Bob']

// Validator factory
function validator(rules) {
  return function (data) {
    return rules.every((rule) => rule(data));
  };
}

const isValidUser = validator([
  (user) => user.name && user.name.length > 0,
  (user) => user.age >= 18,
  (user) => user.email && user.email.includes("@"),
]);
```

### **Example 4: Partial Application**

```javascript
function partial(fn, ...fixedArgs) {
  return function (...remainingArgs) {
    return fn(...fixedArgs, ...remainingArgs);
  };
}

// Example: logging with timestamp
function log(level, message) {
  console.log(`[${level}] ${message}`);
}

const error = partial(log, "ERROR");
const warn = partial(log, "WARN");
const info = partial(log, "INFO");

error("Something went wrong"); // [ERROR] Something went wrong
warn("This is a warning"); // [WARN] This is a warning
```

---

## 🏗️ Best Practices

1. **Use built-in HOFs over loops** - More declarative and functional

   ```javascript
   // ✅ Good
   const doubled = numbers.map((n) => n * 2);

   // ❌ Avoid
   const doubled = [];
   for (const n of numbers) {
     doubled.push(n * 2);
   }
   ```

2. **Chain operations for readability** - Each step is clear

   ```javascript
   users
     .filter((u) => u.active)
     .map((u) => u.name)
     .sort();
   ```

3. **Keep callback functions pure** - No side effects

   ```javascript
   // ✅ Good
   const doubled = numbers.map((n) => n * 2);

   // ❌ Bad
   const results = [];
   numbers.map((n) => results.push(n * 2)); // Side effect
   ```

4. **Use descriptive names** - Clarity over brevity

   ```javascript
   // ✅ Good
   const activeUsers = users.filter((user) => user.active);

   // ❌ Less clear
   const x = users.filter((u) => u.a);
   ```

---

## 🧪 Interview Questions

### **Q1: What is a higher-order function?**

**Answer:** A higher-order function either:

1. Takes one or more functions as arguments
2. Returns a function

Examples: `map`, `filter`, `reduce`, `setTimeout`, `addEventListener`.

### **Q2: Implement map using reduce.**

**Answer:**

```javascript
function map(array, fn) {
  return array.reduce((acc, item) => {
    return [...acc, fn(item)];
  }, []);
}
```

### **Q3: What's the difference between map and forEach?**

**Answer:**

- **map** returns a new array with transformed elements
- **forEach** returns undefined, used only for side effects

```javascript
const doubled = [1, 2, 3].map((n) => n * 2); // [2, 4, 6]
[1, 2, 3].forEach((n) => console.log(n)); // undefined
```

Prefer `map` for transformations, `forEach` only for side effects.

---

## 📚 Key Takeaways

- Higher-order functions take functions as arguments or return functions
- Built-in HOFs: map (transform), filter (select), reduce (aggregate)
- HOFs enable code reuse, abstraction, and declarative programming
- Chain HOFs for readable data transformations
- Keep callback functions pure (no side effects)
- Use HOFs over imperative loops for cleaner, more functional code
- HOFs are the foundation of functional composition

---

**Master higher-order functions for production-ready code!**
