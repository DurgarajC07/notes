# Memory Management

> **Heap, stack, garbage collection, and memory leaks**

---

## 🎯 Overview

JavaScript's automatic memory management handles allocation and deallocation through garbage collection. Understanding the heap, stack, garbage collection algorithms, and how to prevent memory leaks is essential for building performant applications.

---

## 📚 Core Concepts

### **Concept 1: Memory Structure**

JavaScript uses two memory regions:

**Stack Memory:**

- Stores primitive values and function execution contexts
- Fixed size, fast allocation/deallocation (LIFO)
- Automatically managed when functions return

**Heap Memory:**

- Stores objects, arrays, and closures
- Dynamic size, slower than stack
- Managed by garbage collector

```javascript
// Stack memory
function calculate() {
  const x = 5; // Primitive on stack
  const y = 10; // Primitive on stack
  return x + y; // Result on stack
} // x and y popped from stack

// Heap memory
function createUser() {
  const user = {
    // Object on heap
    name: "Alice", // String on heap
    age: 30,
  };
  return user; // Reference on stack, object on heap
}

const myUser = createUser(); // myUser is reference on stack
```

### **Concept 2: Garbage Collection**

**Mark and Sweep Algorithm:**

V8 uses the mark-and-sweep algorithm:

1. **Mark Phase:** Starting from roots (global objects, stack), mark all reachable objects
2. **Sweep Phase:** Remove unmarked (unreachable) objects

```javascript
// Example: Object becomes unreachable
let user = { name: "Alice" }; // user references object
user = null; // Object no longer reachable - eligible for GC

// Circular references are handled
let obj1 = {};
let obj2 = {};
obj1.ref = obj2;
obj2.ref = obj1;
obj1 = null;
obj2 = null; // Both objects unreachable - GC collects them
```

**Generational Garbage Collection:**

V8 divides heap into generations:

- **Young Generation (Scavenger):** Short-lived objects, collected frequently
  - **Nursery:** New allocations
  - **Intermediate:** Survived one GC
- **Old Generation (Mark-Sweep-Compact):** Long-lived objects, collected less frequently

```javascript
// Young generation
function temp() {
  const arr = [1, 2, 3]; // Allocated in young generation
  return arr.reduce((a, b) => a + b);
} // arr becomes unreachable - quickly collected

// Old generation
const globalCache = new Map(); // Lives long - moves to old generation
```

### **Concept 3: Memory Leaks**

Common causes of memory leaks:

**1. Global Variables:**

```javascript
// ❌ Accidental global
function createLeak() {
  leak = "I am global"; // Missing var/let/const
}

// ✅ Use strict mode
("use strict");
function noLeak() {
  // leak = 'Error'; // ReferenceError
  const local = "Safe";
}
```

**2. Forgotten Timers:**

```javascript
// ❌ Leak - timer never cleared
function startTimer() {
  const data = new Array(1000000);
  setInterval(() => {
    console.log(data[0]); // data never released
  }, 1000);
}

// ✅ Clear timers
function startTimer() {
  const data = new Array(1000000);
  const timerId = setInterval(() => {
    console.log(data[0]);
  }, 1000);

  return () => clearInterval(timerId); // Cleanup function
}

const cleanup = startTimer();
// Later: cleanup();
```

**3. Event Listeners:**

```javascript
// ❌ Leak - listener not removed
function attachListener() {
  const element = document.getElementById("button");
  const data = new Array(1000000);

  element.addEventListener("click", () => {
    console.log(data[0]); // Keeps data in memory
  });
}

// ✅ Remove listeners
function attachListener() {
  const element = document.getElementById("button");
  const data = new Array(1000000);

  const handler = () => console.log(data[0]);
  element.addEventListener("click", handler);

  return () => element.removeEventListener("click", handler);
}
```

**4. Closures:**

```javascript
// ❌ Unintentional retention
function createHandler() {
  const largeData = new Array(1000000).fill("data");

  return {
    getFirst: () => largeData[0], // Keeps entire array
    getLast: () => largeData[largeData.length - 1],
  };
}

// ✅ Store only what's needed
function createHandler() {
  const largeData = new Array(1000000).fill("data");
  const first = largeData[0];
  const last = largeData[largeData.length - 1];

  return {
    getFirst: () => first, // largeData can be GC'd
    getLast: () => last,
  };
}
```

**5. Detached DOM Nodes:**

```javascript
// ❌ Leak - DOM node referenced after removal
let detachedNodes = [];

function createNodes() {
  const parent = document.getElementById("container");
  const child = document.createElement("div");
  parent.appendChild(child);

  detachedNodes.push(child); // Keeps reference
  parent.removeChild(child); // Removed from DOM but still in memory
}

// ✅ Clear references
function createNodes() {
  const parent = document.getElementById("container");
  const child = document.createElement("div");
  parent.appendChild(child);

  return () => {
    parent.removeChild(child);
    // No lingering references - can be GC'd
  };
}
```

---

## 🔥 Practical Examples

### **Example 1: WeakMap for Cache**

Use WeakMap to avoid memory leaks with object keys:

```javascript
// ❌ Regular Map - memory leak
const cache = new Map();

function process(obj) {
  if (!cache.has(obj)) {
    cache.set(obj, expensiveOperation(obj));
  }
  return cache.get(obj);
}

// Objects remain in cache even if not used elsewhere

// ✅ WeakMap - no leak
const cache = new WeakMap();

function process(obj) {
  if (!cache.has(obj)) {
    cache.set(obj, expensiveOperation(obj));
  }
  return cache.get(obj);
}

// When obj is no longer referenced elsewhere,
// it's automatically removed from WeakMap
```

### **Example 2: Proper Event Listener Management**

```javascript
class Component {
  constructor(element) {
    this.element = element;
    this.handlers = new Map();
  }

  addEventListener(event, handler) {
    this.element.addEventListener(event, handler);

    if (!this.handlers.has(event)) {
      this.handlers.set(event, []);
    }
    this.handlers.get(event).push(handler);
  }

  destroy() {
    // Clean up all event listeners
    for (const [event, handlers] of this.handlers) {
      handlers.forEach((handler) => {
        this.element.removeEventListener(event, handler);
      });
    }
    this.handlers.clear();
  }
}

// Usage
const component = new Component(document.getElementById("app"));
component.addEventListener("click", handleClick);

// When done
component.destroy(); // No memory leak
```

### **Example 3: AbortController for Fetch**

```javascript
// ❌ Potential leak - pending request holds memory
async function fetchUser(id) {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}

// Component unmounts but fetch continues

// ✅ Use AbortController
class UserFetcher {
  constructor() {
    this.abortController = null;
  }

  async fetchUser(id) {
    // Cancel previous request
    if (this.abortController) {
      this.abortController.abort();
    }

    this.abortController = new AbortController();

    try {
      const response = await fetch(`/api/users/${id}`, {
        signal: this.abortController.signal,
      });
      return await response.json();
    } catch (error) {
      if (error.name === "AbortError") {
        console.log("Request cancelled");
      }
      throw error;
    }
  }

  cleanup() {
    if (this.abortController) {
      this.abortController.abort();
    }
  }
}
```

### **Example 4: Memory-Efficient Data Processing**

```javascript
// ❌ Loads entire file into memory
async function processLargeFile() {
  const response = await fetch("/large-file.json");
  const data = await response.json(); // All in memory
  return data.map((item) => transform(item));
}

// ✅ Stream processing
async function processLargeFile() {
  const response = await fetch("/large-file.json");
  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  let buffer = "";
  const results = [];

  while (true) {
    const { done, value } = await reader.read();

    if (done) break;

    buffer += decoder.decode(value, { stream: true });

    // Process complete items
    const lines = buffer.split("\n");
    buffer = lines.pop(); // Keep incomplete line

    for (const line of lines) {
      if (line.trim()) {
        results.push(transform(JSON.parse(line)));
      }
    }
  }

  return results;
}
```

---

## 🏗️ Best Practices

1. **Use WeakMap/WeakSet for metadata** - Prevents memory leaks with object keys

   ```javascript
   const metadata = new WeakMap();
   metadata.set(obj, { created: Date.now() });
   ```

2. **Always clean up resources** - Remove event listeners, clear timers, abort requests

   ```javascript
   componentWillUnmount() {
     this.cleanup();
   }
   ```

3. **Avoid global variables** - Use modules and closures instead

4. **Nullify references** - Help GC identify unreachable objects

   ```javascript
   let largeObject = createLargeObject();
   // ... use it ...
   largeObject = null; // Eligible for GC
   ```

5. **Use Chrome DevTools** - Profile memory usage and find leaks
   - Heap snapshots
   - Allocation timeline
   - Memory profiler

---

## 🧪 Interview Questions

### **Q1: Explain JavaScript's garbage collection.**

**Answer:** JavaScript uses automatic garbage collection with mark-and-sweep algorithm. V8 employs generational GC: young generation (scavenger) for short-lived objects collected frequently, and old generation (mark-sweep-compact) for long-lived objects collected less often. Objects are marked as reachable from roots (global, stack), and unreachable objects are swept away.

### **Q2: What causes memory leaks in JavaScript?**

**Answer:** Common causes include:

1. Forgotten event listeners
2. Uncleaned timers/intervals
3. Closures retaining large data structures
4. Detached DOM nodes still referenced
5. Accidental globals
6. Growing caches without limits

Prevention: Always clean up resources, use WeakMap/WeakSet, remove listeners, clear timers.

### **Q3: Difference between WeakMap and Map?**

**Answer:**

- **Map:** Keys can be any type; prevents garbage collection of key objects; iterable
- **WeakMap:** Keys must be objects; allows garbage collection of keys when no other references exist; not iterable

Use WeakMap for object metadata to prevent memory leaks.

---

## 📚 Key Takeaways

- JavaScript uses stack (primitives, function contexts) and heap (objects, arrays)
- V8's garbage collector uses generational mark-and-sweep algorithm
- Common memory leaks: forgotten listeners, timers, closures, detached DOM
- Use WeakMap/WeakSet to prevent leaks with object keys
- Always clean up: remove listeners, clear timers, abort requests
- Profile with Chrome DevTools to identify and fix memory issues

---

**Master memory management for production-ready code!**
