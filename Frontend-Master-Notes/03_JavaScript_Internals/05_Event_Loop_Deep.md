# Event Loop Deep Dive

> **Macrotasks, microtasks, and execution order**

---

## 🎯 Overview

The Event Loop is JavaScript's concurrency model that handles asynchronous operations. Understanding macrotasks, microtasks, and their execution order is crucial for debugging timing issues and optimizing performance.

---

## 📚 Core Concepts

### **Concept 1: Event Loop Architecture**

```
Call Stack
    ↓
Microtask Queue (Promise callbacks, queueMicrotask)
    ↓
Macrotask Queue (setTimeout, setInterval, I/O)
    ↓
Render (if browser)
```

**Execution order:**

1. Execute all synchronous code (call stack)
2. Execute all microtasks (until queue is empty)
3. Execute one macrotask
4. Render (browser only)
5. Repeat from step 2

### **Concept 2: Microtasks vs Macrotasks**

**Microtasks** (higher priority):

- Promise callbacks (`.then`, `.catch`, `.finally`)
- `queueMicrotask()`
- `MutationObserver` callbacks
- `process.nextTick()` (Node.js)

**Macrotasks** (lower priority):

- `setTimeout`, `setInterval`
- `setImmediate` (Node.js)
- I/O operations
- UI rendering
- `requestAnimationFrame` (browser)

```javascript
console.log("1: Sync");

setTimeout(() => console.log("2: Macrotask"), 0);

Promise.resolve().then(() => console.log("3: Microtask"));

console.log("4: Sync");

// Output:
// 1: Sync
// 4: Sync
// 3: Microtask  ← Microtasks run first
// 2: Macrotask
```

### **Concept 3: Microtask Queue Draining**

Microtasks run until queue is empty:

```javascript
console.log("Start");

setTimeout(() => console.log("Timeout"), 0);

Promise.resolve()
  .then(() => {
    console.log("Promise 1");
    return Promise.resolve();
  })
  .then(() => console.log("Promise 2"));

Promise.resolve().then(() => console.log("Promise 3"));

console.log("End");

// Output:
// Start
// End
// Promise 1
// Promise 3  ← All microtasks before macrotask
// Promise 2
// Timeout
```

---

## 🔥 Practical Examples

### **Example 1: Execution Order Puzzle**

```javascript
console.log("1");

setTimeout(() => {
  console.log("2");
  Promise.resolve().then(() => console.log("3"));
}, 0);

Promise.resolve()
  .then(() => {
    console.log("4");
    setTimeout(() => console.log("5"), 0);
  })
  .then(() => console.log("6"));

console.log("7");

// Output:
// 1 (sync)
// 7 (sync)
// 4 (microtask)
// 6 (microtask)
// 2 (macrotask 1)
// 3 (microtask from macrotask 1)
// 5 (macrotask 2)
```

### **Example 2: Long Microtask Queue**

```javascript
function infiniteMicrotasks() {
  Promise.resolve().then(infiniteMicrotasks);
}

infiniteMicrotasks();

// Macrotasks never run - UI freezes!
// Microtask queue never empties
```

### **Example 3: requestAnimationFrame Timing**

```javascript
console.log("1: Sync");

setTimeout(() => console.log("2: Timeout"), 0);

Promise.resolve().then(() => console.log("3: Promise"));

requestAnimationFrame(() => console.log("4: rAF"));

console.log("5: Sync");

// Output (browser):
// 1: Sync
// 5: Sync
// 3: Promise     ← Microtasks
// 4: rAF         ← Before next macrotask
// 2: Timeout
```

### **Example 4: Breaking Long Tasks**

```javascript
// ❌ Blocks event loop
function processLargeArray(items) {
  items.forEach((item) => {
    heavyProcessing(item);
  });
}

// ✅ Allows event loop to breathe
async function processLargeArray(items) {
  for (const item of items) {
    heavyProcessing(item);

    // Yield to event loop every 100 items
    if (items.indexOf(item) % 100 === 0) {
      await new Promise((resolve) => setTimeout(resolve, 0));
    }
  }
}

// ✅ Better: Use chunks
async function processInChunks(items, chunkSize = 100) {
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    chunk.forEach((item) => heavyProcessing(item));
    await new Promise((resolve) => setTimeout(resolve, 0));
  }
}
```

---

## 🏗️ Best Practices

1. **Avoid long microtask chains** - Can block rendering

   ```javascript
   // Break into macrotasks if needed
   setTimeout(() => continueLongTask(), 0);
   ```

2. **Use queueMicrotask for critical updates** - Higher priority than setTimeout

   ```javascript
   queueMicrotask(() => {
     updateCriticalState();
   });
   ```

3. **Chunk heavy operations** - Prevent blocking the main thread
   ```javascript
   async function processHeavyWork(items) {
     for (let i = 0; i < items.length; i++) {
       processItem(items[i]);
       if (i % 100 === 0) {
         await new Promise((r) => setTimeout(r, 0));
       }
     }
   }
   ```

---

## 🧪 Interview Questions

### **Q1: Explain the difference between microtasks and macrotasks.**

**Answer:**
**Microtasks** (Promise callbacks, queueMicrotask) have higher priority and run completely before the next macrotask. **Macrotasks** (setTimeout, I/O) run one per event loop cycle. Order: sync code → all microtasks → one macrotask → render → repeat.

Example:

```javascript
setTimeout(() => console.log("macro"), 0);
Promise.resolve().then(() => console.log("micro"));
// Output: micro, macro
```

### **Q2: What happens if microtasks keep queuing more microtasks?**

**Answer:** The microtask queue never empties, blocking macrotasks and UI rendering. This causes the page to freeze. Always ensure microtask chains terminate.

```javascript
function infinite() {
  Promise.resolve().then(infinite); // UI freezes!
}
```

---

## 📚 Key Takeaways

- Event loop processes: sync code → microtasks → one macrotask → render → repeat
- Microtasks (Promises) have higher priority than macrotasks (setTimeout)
- All microtasks run before the next macrotask
- Long microtask chains can block rendering and freeze UI
- Break heavy work into chunks using setTimeout to yield to event loop
- Understanding execution order is crucial for debugging async code

---

**Master event loop deep dive for production-ready code!**
