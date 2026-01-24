# JavaScript Optimization Techniques

> **Performance optimization and avoiding deoptimization**

---

## 🎯 Overview

JavaScript optimization involves understanding how engines like V8 optimize code and avoiding patterns that cause deoptimization. Writing optimization-friendly code can dramatically improve application performance.

---

## 📚 Core Concepts

### **Concept 1: Monomorphic vs Polymorphic Code**

**Monomorphic** code works with one shape/type (fastest):

```javascript
// ✅ Monomorphic - always receives same shape
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
}

function distance(point) {
  return Math.sqrt(point.x ** 2 + point.y ** 2);
}

distance(new Point(3, 4)); // Optimized for Point shape
distance(new Point(5, 12)); // Same shape - stays monomorphic
```

**Polymorphic** code works with multiple shapes (slower):

```javascript
// ⚠️ Polymorphic - handles different shapes
function getValue(obj) {
  return obj.value;
}

getValue({ value: 1 }); // Shape 1
getValue({ value: 2, extra: "data" }); // Shape 2
getValue({ other: "prop", value: 3 }); // Shape 3
// Function becomes polymorphic - slower
```

**Megamorphic** code works with many shapes (slowest):

```javascript
// ❌ Megamorphic - too many shapes
function getProperty(obj, key) {
  return obj[key]; // Dynamic property access
}

// Called with dozens of different object shapes
// Becomes megamorphic - very slow
```

### **Concept 2: Inline Caching Optimization**

Write code that enables inline caching:

```javascript
// ✅ Good for IC - consistent object shape
function processUsers(users) {
  return users.map((user) => ({
    name: user.name,
    email: user.email,
  }));
}

// All users have same shape - IC optimizes property access

// ❌ Bad for IC - inconsistent shapes
function processItems(items) {
  return items.map((item) => {
    // Items have different properties
    if (item.type === "user") return item.name;
    if (item.type === "post") return item.title;
    return item.label;
  });
}
```

### **Concept 3: Hidden Class Stability**

Maintain stable hidden classes:

```javascript
// ❌ Hidden class instability
function createUser(name, age) {
  const user = {};
  if (name) user.name = name; // Conditional property
  if (age) user.age = age;
  return user;
}

// Creates multiple hidden classes:
// {} → {name} or {age} or {name, age}

// ✅ Stable hidden class
function createUser(name = null, age = null) {
  return { name, age }; // Always same shape
}

// One hidden class for all users
```

---

## 🔥 Practical Examples

### **Example 1: Array Optimization**

```javascript
// ❌ Deoptimized array - mixed types
const mixed = [1, "two", 3, "four"];
// PACKED_ELEMENTS - slower

// ✅ Optimized array - consistent type
const numbers = [1, 2, 3, 4];
// PACKED_SMI_ELEMENTS - fastest

// ❌ Sparse array - holes
const sparse = new Array(1000);
sparse[0] = 1;
sparse[999] = 2;
// HOLEY_ELEMENTS - slowest

// ✅ Dense array - no holes
const dense = new Array(1000).fill(0);
dense[0] = 1;
dense[999] = 2;
// PACKED_ELEMENTS - much faster
```

### **Example 2: Function Optimization**

```javascript
// ❌ Hard to optimize
function calculate(a, b, op) {
  switch (op) {
    case "add":
      return a + b;
    case "sub":
      return a - b;
    case "mul":
      return a * b;
    case "div":
      return a / b;
  }
}

// ✅ Easy to optimize - separate functions
const operations = {
  add: (a, b) => a + b,
  sub: (a, b) => a - b,
  mul: (a, b) => a * b,
  div: (a, b) => a / b,
};

function calculate(a, b, op) {
  return operations[op](a, b);
}
// Each operation function stays monomorphic
```

### **Example 3: Object Property Access**

```javascript
// ❌ Slow - dynamic property access
function getProperty(obj, key) {
  return obj[key]; // V8 can't optimize
}

// ✅ Fast - static property access
function getName(obj) {
  return obj.name; // V8 optimizes with IC
}

function getEmail(obj) {
  return obj.email;
}

// ✅ Better - property map for known properties
const getters = {
  name: (obj) => obj.name,
  email: (obj) => obj.email,
  age: (obj) => obj.age,
};

function getProperty(obj, key) {
  return getters[key]?.(obj);
}
```

### **Example 4: Avoiding Deoptimization**

```javascript
// ❌ Causes deopt - arguments object
function sum() {
  let total = 0;
  for (let i = 0; i < arguments.length; i++) {
    total += arguments[i];
  }
  return total;
}

// ✅ Optimizable - rest parameters
function sum(...numbers) {
  let total = 0;
  for (let i = 0; i < numbers.length; i++) {
    total += numbers[i];
  }
  return total;
}

// ❌ Causes deopt - try/catch with variable
function parse(str) {
  let result;
  try {
    result = JSON.parse(str);
  } catch (e) {
    result = null;
  }
  return result;
}

// ✅ Optimizable - isolate try/catch
function parse(str) {
  return trySafeParse(str) || null;
}

function trySafeParse(str) {
  try {
    return JSON.parse(str);
  } catch {
    return null;
  }
}
```

### **Example 5: Loop Optimization**

```javascript
// ❌ Suboptimal
const arr = [1, 2, 3, 4, 5];
for (let i = 0; i < arr.length; i++) {
  // arr.length accessed every iteration
  console.log(arr[i]);
}

// ✅ Optimized - cache length
const arr = [1, 2, 3, 4, 5];
const len = arr.length;
for (let i = 0; i < len; i++) {
  console.log(arr[i]);
}

// ✅ Even better - for...of (built-in optimization)
const arr = [1, 2, 3, 4, 5];
for (const item of arr) {
  console.log(item);
}
```

---

## 🏗️ Best Practices

1. **Keep functions monomorphic** - Same input shapes, same types

   ```javascript
   function process(point) {
     /* always Point */
   }
   ```

2. **Initialize all properties** - Avoid changing object shape

   ```javascript
   class User {
     constructor() {
       this.name = null;
       this.email = null;
     }
   }
   ```

3. **Use consistent types** - Don't mix numbers and strings

   ```javascript
   const ids = [1, 2, 3]; // All numbers
   ```

4. **Avoid sparse arrays** - Use dense arrays

   ```javascript
   const arr = new Array(10).fill(0); // Not new Array(10)
   ```

5. **Minimize property deletion** - Set to null instead

   ```javascript
   obj.prop = null; // Not delete obj.prop
   ```

6. **Cache array length** - Or use for...of

   ```javascript
   const len = arr.length;
   ```

7. **Isolate deoptimizing code** - Keep try/catch separate
   ```javascript
   function safeParse(str) {
     try {
       return JSON.parse(str);
     } catch {
       return null;
     }
   }
   ```

---

## 🧪 Interview Questions

### **Q1: What is the difference between monomorphic and polymorphic code?**

**Answer:** Monomorphic code works with one object shape/type and is highly optimized by V8 through inline caching. Polymorphic code handles multiple shapes (2-4) and is slower due to shape checks. Megamorphic code handles many shapes and falls back to hash table lookups. Keep functions monomorphic for best performance.

### **Q2: How do you avoid deoptimization?**

**Answer:** Avoid:

- Using `arguments` object (use rest parameters)
- Changing function argument types
- Adding/deleting properties after creation
- try/catch blocks in hot code (isolate them)
- Sparse arrays and type mixing

Instead: Initialize all properties, use consistent types, keep functions monomorphic.

---

## 📚 Key Takeaways

- Monomorphic code (one shape) is fastest, polymorphic slower, megamorphic slowest
- Keep object shapes stable - initialize all properties upfront
- Use consistent types in arrays and function parameters
- Avoid sparse arrays - use dense arrays with fill()
- Isolate try/catch blocks to prevent deoptimization
- Cache array lengths or use for...of loops
- Profile with Chrome DevTools to identify optimization issues

---

**Master javascript optimization for production-ready code!**
