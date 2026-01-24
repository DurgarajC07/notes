# Function Composition

> **Combining simple functions to build complex behavior**

---

## 🎯 Overview

Function composition is the process of combining two or more functions to produce a new function. It's a fundamental concept in functional programming that enables building complex operations from simple, reusable pieces.

---

## 📚 Core Concepts

### **Concept 1: Basic Composition**

Composition combines functions: `(f ∘ g)(x) = f(g(x))`

```javascript
// Manual composition
const add1 = (x) => x + 1;
const double = (x) => x * 2;

// Compose manually
const add1ThenDouble = (x) => double(add1(x));
add1ThenDouble(3); // double(add1(3)) = double(4) = 8

// Generic compose function
function compose(f, g) {
  return function (x) {
    return f(g(x));
  };
}

const add1ThenDouble = compose(double, add1);
add1ThenDouble(3); // 8
```

**Direction:** Compose executes right-to-left (mathematical notation):

```javascript
compose(f, g, h)(x) === f(g(h(x)));
```

### **Concept 2: Compose vs Pipe**

**Compose** - Right-to-left execution:

```javascript
function compose(...fns) {
  return function (x) {
    return fns.reduceRight((acc, fn) => fn(acc), x);
  };
}

// compose(f, g, h)(x) = f(g(h(x)))
const transform = compose(
  toUpperCase, // 3. Last
  trim, // 2. Second
  addPrefix, // 1. First
);

transform("hello"); // 'PREFIX:HELLO'
```

**Pipe** - Left-to-right execution (more intuitive):

```javascript
function pipe(...fns) {
  return function (x) {
    return fns.reduce((acc, fn) => fn(acc), x);
  };
}

// pipe(f, g, h)(x) = h(g(f(x)))
const transform = pipe(
  addPrefix, // 1. First
  trim, // 2. Second
  toUpperCase, // 3. Last
);

transform("hello"); // 'PREFIX:HELLO'
```

### **Concept 3: Point-Free Style**

Defining functions without mentioning arguments:

```javascript
// With points (arguments)
const getActiveUserNames = (users) =>
  users.filter((user) => user.active).map((user) => user.name);

// Point-free style
const isActive = (user) => user.active;
const getName = (user) => user.name;

const getActiveUserNames = pipe(filter(isActive), map(getName));

// Even more point-free
const prop = (key) => (obj) => obj[key];
const getActiveUserNames = pipe(filter(prop("active")), map(prop("name")));
```

---

## 🔥 Practical Examples

### **Example 1: Data Transformation Pipeline**

```javascript
// Helper functions
const trim = (str) => str.trim();
const toLowerCase = (str) => str.toLowerCase();
const removeSpaces = (str) => str.replace(/\s+/g, "");
const addPrefix = (prefix) => (str) => `${prefix}${str}`;

// Compose a slug generator
const slugify = pipe(
  trim,
  toLowerCase,
  (str) => str.replace(/\s+/g, "-"),
  (str) => str.replace(/[^a-z0-9-]/g, ""),
);

sluggify("  Hello World! 123  "); // 'hello-world-123'
```

### **Example 2: Validation Pipeline**

```javascript
// Validators
const required = (field) => (value) => {
  if (!value) return `${field} is required`;
  return null;
};

const minLength = (field, min) => (value) => {
  if (value.length < min) return `${field} must be at least ${min} characters`;
  return null;
};

const isEmail = (field) => (value) => {
  if (!value.includes("@")) return `${field} must be a valid email`;
  return null;
};

// Compose validators
function validate(...validators) {
  return (value) => {
    for (const validator of validators) {
      const error = validator(value);
      if (error) return { valid: false, error };
    }
    return { valid: true };
  };
}

const validateEmail = validate(
  required("Email"),
  minLength("Email", 5),
  isEmail("Email"),
);

validateEmail("a@b"); // { valid: false, error: 'Email must be at least 5 characters' }
validateEmail("user@example.com"); // { valid: true }
```

### **Example 3: API Response Processing**

```javascript
// Processing functions
const parseJSON = (response) => response.json();
const getData = (obj) => obj.data;
const filterActive = (items) => items.filter((item) => item.active);
const sortByName = (items) =>
  [...items].sort((a, b) => a.name.localeCompare(b.name));
const mapToDTO = (item) => ({
  id: item.id,
  name: item.name,
  displayName: `${item.name} (${item.id})`,
});

// Compose processing pipeline
const processAPIResponse = pipe(
  parseJSON,
  getData,
  filterActive,
  sortByName,
  (items) => items.map(mapToDTO),
);

// Usage
fetch("/api/users")
  .then(processAPIResponse)
  .then((users) => console.log(users));
```

### **Example 4: Currying for Composition**

```javascript
// Curried functions for better composition
const map = (fn) => (array) => array.map(fn);
const filter = (pred) => (array) => array.filter(pred);
const reduce = (fn, init) => (array) => array.reduce(fn, init);

// Reusable predicates and transformers
const isEven = (n) => n % 2 === 0;
const double = (n) => n * 2;
const sum = (acc, n) => acc + n;

// Compose pipeline
const sumOfDoubledEvens = pipe(filter(isEven), map(double), reduce(sum, 0));

sumOfDoubledEvens([1, 2, 3, 4, 5, 6]);
// filter: [2, 4, 6]
// map: [4, 8, 12]
// reduce: 24
```

### **Example 5: Debugging Compositions**

```javascript
// Tap function for debugging
const tap = (label) => (value) => {
  console.log(label, value);
  return value;
};

const transform = pipe(
  addPrefix(">> "),
  tap("after addPrefix:"),
  trim,
  tap("after trim:"),
  toLowerCase,
  tap("after toLowerCase:"),
);

transform("  Hello  ");
// Logs:
// after addPrefix: >>   Hello
// after trim: >> Hello
// after toLowerCase: >> hello
```

---

## 🏗️ Best Practices

1. **Prefer pipe over compose** - Left-to-right is more intuitive

   ```javascript
   const process = pipe(step1, step2, step3);
   ```

2. **Keep functions small and focused** - Single responsibility

   ```javascript
   const trim = (str) => str.trim();
   const toLowerCase = (str) => str.toLowerCase();
   ```

3. **Use currying for composition** - Makes functions more composable

   ```javascript
   const map = (fn) => (array) => array.map(fn);
   const double = map((n) => n * 2);
   ```

4. **Name intermediate functions** - Self-documenting code

   ```javascript
   const getActiveUsers = filter((u) => u.active);
   const getUserNames = map((u) => u.name);
   const sortNames = sort((a, b) => a.localeCompare(b));

   const process = pipe(getActiveUsers, getUserNames, sortNames);
   ```

5. **Use tap for debugging** - Inspect values in pipeline
   ```javascript
   pipe(transform1, tap("after transform1"), transform2);
   ```

---

## 🧪 Interview Questions

### **Q1: What is function composition?**

**Answer:** Function composition is combining two or more functions to create a new function. Given `f` and `g`, composition `compose(f, g)` returns a function where `compose(f, g)(x) = f(g(x))`.

Benefits: Reusability, testability, readability.

### **Q2: What's the difference between compose and pipe?**

**Answer:**

- **compose** executes right-to-left: `compose(f, g, h)(x) = f(g(h(x)))`
- **pipe** executes left-to-right: `pipe(f, g, h)(x) = h(g(f(x)))`

Pipe is often preferred for better readability.

### **Q3: Implement a pipe function.**

**Answer:**

```javascript
function pipe(...fns) {
  return function (x) {
    return fns.reduce((acc, fn) => fn(acc), x);
  };
}

// Usage
const add1 = (x) => x + 1;
const double = (x) => x * 2;
const subtract5 = (x) => x - 5;

const transform = pipe(add1, double, subtract5);
transform(3); // (3 + 1) * 2 - 5 = 3
```

---

## 📚 Key Takeaways

- Function composition combines simple functions to build complex behavior
- compose executes right-to-left, pipe executes left-to-right
- Prefer pipe for better readability (left-to-right data flow)
- Keep functions small, focused, and pure for better composition
- Currying makes functions more composable
- Point-free style removes explicit argument passing
- Use composition to create reusable, testable data pipelines

---

**Master function composition for production-ready code!**
