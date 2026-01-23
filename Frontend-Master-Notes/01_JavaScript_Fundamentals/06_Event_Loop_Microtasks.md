# ⚡ Event Loop & Microtasks

> The event loop is JavaScript's concurrency model. Understanding it is crucial for writing performant async code, debugging timing issues, and acing senior interviews.

---

## 📖 1. Concept Explanation

### What is the Event Loop?

The **event loop** is how JavaScript handles asynchronous operations despite being single-threaded. It coordinates execution between the call stack, task queues, and microtask queue.

```javascript
console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve().then(() => console.log("3"));

console.log("4");

// Output: 1, 4, 3, 2
// Why? Event loop priority!
```

### Architecture

```
┌─────────────────────────────┐
│     Call Stack              │ ← Currently executing code
└─────────────────────────────┘
              ↓
┌─────────────────────────────┐
│   Microtask Queue           │ ← Promises, queueMicrotask
│   (Higher Priority)         │
└─────────────────────────────┘
              ↓
┌─────────────────────────────┐
│   Task Queue (Macrotasks)   │ ← setTimeout, setInterval
│   (Lower Priority)          │
└─────────────────────────────┘
```

### Key Components

**1. Call Stack:**

- LIFO (Last In, First Out)
- Currently executing code
- One function at a time

**2. Microtask Queue:**

- Promise callbacks (`.then`, `.catch`, `.finally`)
- `queueMicrotask()`
- `MutationObserver`
- Process.nextTick (Node.js)

**3. Task Queue (Macrotasks):**

- `setTimeout`, `setInterval`
- `setImmediate` (Node.js)
- I/O operations
- UI rendering

**4. Web APIs:**

- Browser-provided APIs (setTimeout, fetch, DOM events)
- Execute in separate threads
- Push callbacks to queues when complete

---

## 🧠 2. Why It Matters in Real Projects

### Production Impact

**Timing Bugs:**

```javascript
// ❌ BAD: Race condition
let data = null;

fetchData().then((result) => {
  data = result;
});

console.log(data); // null! Promise hasn't resolved yet
```

**Performance Issues:**

```javascript
// ❌ BAD: Blocking the main thread
function heavyComputation() {
  for (let i = 0; i < 1e9; i++) {
    // Long-running sync operation
  }
  return result;
}

heavyComputation(); // UI freezes!

// ✅ GOOD: Yield to event loop
async function heavyComputationAsync() {
  const chunks = 100;
  for (let i = 0; i < chunks; i++) {
    await new Promise((resolve) => setTimeout(resolve, 0));
    // Process chunk
  }
}
```

### Real-World Scenarios

**1. React State Updates:**

```javascript
// Why setState is async
function handleClick() {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
  // Only updates once! Batched in microtask
}
```

**2. API Calls:**

```javascript
// Promise timing
async function loadData() {
  console.log("1: Start");

  const data = await fetch("/api");
  console.log("3: Data loaded");

  console.log("2: After await"); // Won't execute until fetch completes
}
```

---

## ⚙️ 3. Internal Working

### Event Loop Phases

```javascript
// Event loop algorithm (simplified)
while (true) {
  // 1. Execute all synchronous code (call stack)
  while (callStack.length > 0) {
    execute(callStack.pop());
  }

  // 2. Process ALL microtasks
  while (microtaskQueue.length > 0) {
    execute(microtaskQueue.shift());
  }

  // 3. Render (if needed)
  if (shouldRender()) {
    render();
  }

  // 4. Process ONE macrotask
  if (taskQueue.length > 0) {
    execute(taskQueue.shift());
  }

  // Repeat...
}
```

### Execution Order

```javascript
console.log("1: Sync");

setTimeout(() => {
  console.log("5: Macro task");
}, 0);

Promise.resolve()
  .then(() => {
    console.log("3: Micro 1");
    return Promise.resolve();
  })
  .then(() => {
    console.log("4: Micro 2");
  });

console.log("2: Sync");

// Call stack: 1, 2
// Microtask queue: 3, 4
// Task queue: 5
```

### Visual Timeline

```
Time →
│
├─ [Sync Code] console.log('1')
├─ [Sync Code] setTimeout(...) → Task Queue
├─ [Sync Code] Promise.then(...) → Microtask Queue
├─ [Sync Code] console.log('2')
├─ [Call Stack Empty]
├─ [Process Microtasks] console.log('3')
├─ [Process Microtasks] console.log('4')
├─ [Render if needed]
├─ [Process Task] console.log('5')
└─ [Repeat]
```

### Browser's Rendering Loop

```javascript
// Rendering happens BETWEEN tasks
console.log("1");

// This blocks rendering
for (let i = 0; i < 1e9; i++) {
  // Long loop - UI frozen!
}

console.log("2");

// Break into chunks to allow rendering
async function renderFriendly() {
  for (let i = 0; i < 100; i++) {
    // Work chunk
    await new Promise((r) => setTimeout(r, 0)); // Yield to render
  }
}
```

---

## ✅ 4. Best Practices

### DO ✅

```javascript
// 1. Use microtasks for immediate async work
queueMicrotask(() => {
  // Runs before next task
  updateState();
});

// 2. Break long tasks into chunks
async function processLargeArray(items) {
  const chunkSize = 100;

  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    processChunk(chunk);

    // Yield to event loop
    await new Promise((resolve) => setTimeout(resolve, 0));
  }
}

// 3. Use requestIdleCallback for non-critical work
requestIdleCallback((deadline) => {
  while (deadline.timeRemaining() > 0 && tasks.length > 0) {
    performTask(tasks.pop());
  }
});

// 4. Handle errors in async code
Promise.resolve()
  .then(() => {
    throw new Error("Oops");
  })
  .catch((error) => {
    console.error("Caught:", error);
  });

// 5. Use async/await for readability
async function fetchUserData(id) {
  try {
    const user = await fetch(`/api/users/${id}`);
    const posts = await fetch(`/api/users/${id}/posts`);
    return { user, posts };
  } catch (error) {
    handleError(error);
  }
}
```

### DON'T ❌

```javascript
// ❌ Don't block the event loop
function badLoop() {
  while (true) {
    // Infinite loop - UI frozen forever!
  }
}

// ❌ Don't create microtask loops
function badMicrotask() {
  Promise.resolve().then(() => {
    badMicrotask(); // Infinite microtasks!
  });
  // Tasks never execute!
}

// ❌ Don't rely on setTimeout(0) timing
setTimeout(() => {
  console.log("Not guaranteed to run immediately");
}, 0);
// Minimum 4ms delay in browsers

// ❌ Don't mix sync/async without understanding
function confusing() {
  let value = 1;

  Promise.resolve().then(() => {
    value = 2;
  });

  console.log(value); // Still 1!
}

// ❌ Don't forget error handling
Promise.resolve().then(() => {
  throw new Error("Uncaught!");
});
// Error not caught - goes to window.onerror
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Assuming setTimeout(0) Runs Immediately

```javascript
// ❌ WRONG
console.log("1");

setTimeout(() => {
  console.log("2");
}, 0);

console.log("3");

// Expected: 1, 2, 3
// Actual: 1, 3, 2

// ✅ CORRECT understanding
// setTimeout is a task (macrotask)
// Runs AFTER all sync code and microtasks
```

### Mistake #2: Not Understanding Microtask Priority

```javascript
// ❌ WRONG assumption
setTimeout(() => console.log("1"), 0);
Promise.resolve().then(() => console.log("2"));
setTimeout(() => console.log("3"), 0);

// Output: 2, 1, 3 (not 1, 2, 3)
// Microtasks (Promise) execute before tasks (setTimeout)

// ✅ Priority order:
// 1. Sync code
// 2. ALL microtasks
// 3. ONE task
// 4. Render
// 5. Repeat
```

### Mistake #3: Creating Microtask Starvation

```javascript
// ❌ WRONG: Infinite microtasks starve tasks
function badPattern() {
  Promise.resolve().then(() => {
    console.log("Microtask");
    badPattern(); // Never lets tasks run!
  });
}

badPattern();

setTimeout(() => {
  console.log("This never runs!");
}, 0);

// ✅ CORRECT: Limit microtask recursion
function goodPattern(count = 0) {
  if (count > 100) return; // Limit

  Promise.resolve().then(() => {
    console.log("Microtask", count);
    goodPattern(count + 1);
  });
}
```

### Mistake #4: Race Conditions with Async Code

```javascript
// ❌ WRONG: Race condition
let data = null;

async function loadData() {
  const response = await fetch("/api");
  data = response;
}

loadData();
processData(data); // null! Async hasn't completed

// ✅ CORRECT: Wait for async
async function main() {
  await loadData();
  processData(data); // Now data is loaded
}

// Or use callbacks
loadData().then(() => {
  processData(data);
});
```

---

## 🔐 6. Security Considerations

### Timing Attacks

```javascript
// ❌ VULNERABLE: Observable timing differences
function authenticate(password) {
  const correct = "secret123";

  for (let i = 0; i < password.length; i++) {
    if (password[i] !== correct[i]) {
      return false; // Returns early - timing leak!
    }
  }

  return password.length === correct.length;
}

// ✅ SAFE: Constant-time comparison
function authenticateSafe(password) {
  const correct = "secret123";
  let isValid = password.length === correct.length;

  const maxLength = Math.max(password.length, correct.length);

  for (let i = 0; i < maxLength; i++) {
    const a = password.charCodeAt(i) || 0;
    const b = correct.charCodeAt(i) || 0;
    isValid = isValid && a === b;
  }

  return isValid;
}
```

### Rate Limiting

```javascript
// Prevent event loop flooding
class RateLimiter {
  constructor(maxCalls, windowMs) {
    this.maxCalls = maxCalls;
    this.windowMs = windowMs;
    this.calls = [];
  }

  async execute(fn) {
    const now = Date.now();

    // Remove old calls
    this.calls = this.calls.filter((t) => now - t < this.windowMs);

    if (this.calls.length >= this.maxCalls) {
      const waitTime = this.windowMs - (now - this.calls[0]);
      await new Promise((resolve) => setTimeout(resolve, waitTime));
    }

    this.calls.push(Date.now());
    return fn();
  }
}
```

---

## 🚀 7. Performance Optimization

### 1. Batch Updates

```javascript
// ❌ SLOW: Multiple reflows
function updateUI(items) {
  items.forEach((item) => {
    document.getElementById(item.id).innerText = item.value;
    // Reflow after each update!
  });
}

// ✅ FAST: Batch with microtask
function updateUIBatched(items) {
  queueMicrotask(() => {
    items.forEach((item) => {
      document.getElementById(item.id).innerText = item.value;
    });
    // Single reflow after all updates
  });
}
```

### 2. Yield to Main Thread

```javascript
// ❌ BLOCKS UI
function processLargeDataset(data) {
  return data.map((item) => expensiveOperation(item));
}

// ✅ YIELDS TO UI
async function processLargeDatasetAsync(data) {
  const results = [];

  for (let i = 0; i < data.length; i++) {
    results.push(expensiveOperation(data[i]));

    // Yield every 100 items
    if (i % 100 === 0) {
      await new Promise((resolve) => setTimeout(resolve, 0));
    }
  }

  return results;
}
```

### 3. Use requestAnimationFrame for Animations

```javascript
// ❌ JANKY: setTimeout for animation
function animateBad() {
  setTimeout(() => {
    element.style.left = position + "px";
    if (position < 100) animateBad();
  }, 16); // ~60fps, but not synced with display
}

// ✅ SMOOTH: requestAnimationFrame
function animateGood() {
  requestAnimationFrame(() => {
    element.style.left = position + "px";
    if (position < 100) animateGood();
  });
  // Synced with display refresh rate
}
```

---

## 🧪 8. Code Examples

### Example 1: Task vs Microtask Priority

```javascript
console.log("Script start");

setTimeout(() => {
  console.log("setTimeout");
}, 0);

Promise.resolve()
  .then(() => {
    console.log("Promise 1");
  })
  .then(() => {
    console.log("Promise 2");
  });

console.log("Script end");

// Output:
// Script start
// Script end
// Promise 1
// Promise 2
// setTimeout
```

### Example 2: Nested Promises and setTimeout

```javascript
setTimeout(() => console.log("1"), 0);

Promise.resolve()
  .then(() => {
    console.log("2");
    setTimeout(() => console.log("3"), 0);
  })
  .then(() => {
    console.log("4");
  });

setTimeout(() => console.log("5"), 0);

// Output: 2, 4, 1, 5, 3
// Explanation:
// 1. Sync code completes
// 2. Microtasks: 2, 4 (all microtasks first)
// 3. Task: 1 (first setTimeout)
// 4. Task: 5 (second setTimeout)
// 5. Task: 3 (setTimeout from Promise callback)
```

### Example 3: Chunked Processing

```javascript
async function processLargeArray(array, chunkSize = 100) {
  const results = [];

  for (let i = 0; i < array.length; i += chunkSize) {
    const chunk = array.slice(i, i + chunkSize);

    // Process chunk
    const chunkResults = chunk.map((item) => processItem(item));
    results.push(...chunkResults);

    // Yield to event loop
    await new Promise((resolve) => setTimeout(resolve, 0));

    // Update progress
    const progress = ((i + chunkSize) / array.length) * 100;
    updateProgressBar(Math.min(progress, 100));
  }

  return results;
}

// Usage
const data = Array(10000)
  .fill(0)
  .map((_, i) => i);
processLargeArray(data).then((results) => {
  console.log("Processing complete");
});
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: React State Batching

```javascript
// React batches setState calls in microtasks
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    // These are batched (React 18)
    setCount((c) => c + 1);
    setCount((c) => c + 1);
    setCount((c) => c + 1);

    // Only ONE re-render!
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

### Use Case 2: Debouncing with Microtasks

```javascript
function createDebouncer(delay = 0) {
  let timeoutId = null;
  let microTaskScheduled = false;

  return function debounce(fn) {
    // Cancel previous
    if (timeoutId) clearTimeout(timeoutId);

    // Schedule microtask for immediate debounce
    if (delay === 0 && !microTaskScheduled) {
      microTaskScheduled = true;
      queueMicrotask(() => {
        microTaskScheduled = false;
        fn();
      });
    } else {
      // Schedule task for delayed debounce
      timeoutId = setTimeout(() => {
        timeoutId = null;
        fn();
      }, delay);
    }
  };
}

// Usage
const debouncedSearch = createDebouncer(300);
input.addEventListener("input", () => {
  debouncedSearch(() => search(input.value));
});
```

### Use Case 3: Animation Loop

```javascript
class AnimationController {
  constructor() {
    this.animations = [];
    this.rafId = null;
  }

  add(animation) {
    this.animations.push(animation);
    if (!this.rafId) this.start();
  }

  remove(animation) {
    this.animations = this.animations.filter((a) => a !== animation);
    if (this.animations.length === 0) this.stop();
  }

  start() {
    const loop = (timestamp) => {
      this.animations.forEach((anim) => anim.update(timestamp));
      this.rafId = requestAnimationFrame(loop);
    };

    this.rafId = requestAnimationFrame(loop);
  }

  stop() {
    if (this.rafId) {
      cancelAnimationFrame(this.rafId);
      this.rafId = null;
    }
  }
}

// Usage
const controller = new AnimationController();

const fadeIn = {
  opacity: 0,
  update(timestamp) {
    this.opacity = Math.min(this.opacity + 0.01, 1);
    element.style.opacity = this.opacity;

    if (this.opacity >= 1) {
      controller.remove(this);
    }
  },
};

controller.add(fadeIn);
```

---

## ❓ 10. Interview Questions

### Q1: Explain the event loop in JavaScript.

**Answer:**

The event loop is JavaScript's concurrency model that handles asynchronous operations:

**Components:**

1. **Call Stack** - Executes synchronous code (LIFO)
2. **Microtask Queue** - Promises, queueMicrotask (higher priority)
3. **Task Queue** - setTimeout, setInterval (lower priority)
4. **Web APIs** - Browser APIs running in separate threads

**Execution Order:**

```
1. Execute all synchronous code (call stack)
2. Process ALL microtasks
3. Render (if needed)
4. Process ONE task (macrotask)
5. Repeat
```

**Example:**

```javascript
console.log("1"); // Sync

setTimeout(() => console.log("4"), 0); // Task

Promise.resolve().then(() => console.log("3")); // Microtask

console.log("2"); // Sync

// Output: 1, 2, 3, 4
```

---

### Q2: What's the difference between microtasks and macrotasks?

**Answer:**

| Microtasks                     | Macrotasks              |
| ------------------------------ | ----------------------- |
| Higher priority                | Lower priority          |
| ALL processed before next task | ONE processed per loop  |
| Promise callbacks              | setTimeout, setInterval |
| queueMicrotask()               | I/O, UI events          |
| MutationObserver               | requestAnimationFrame   |

**Key difference:** ALL microtasks execute before ANY task

```javascript
// Microtasks starve tasks
function createMicrotasks() {
  Promise.resolve().then(() => {
    console.log("Microtask");
    createMicrotasks(); // Infinite microtasks!
  });
}

createMicrotasks();

setTimeout(() => {
  console.log("This never runs!"); // Starved
}, 0);
```

---

### Q3: Why does this code produce this output?

```javascript
console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve()
  .then(() => console.log("3"))
  .then(() => console.log("4"));

console.log("5");

// Output: 1, 5, 3, 4, 2
```

**Answer:**

**Execution breakdown:**

1. **Sync code:** `console.log('1')` → Output: 1
2. **setTimeout:** Callback goes to **task queue**
3. **Promise:** First `.then()` goes to **microtask queue**
4. **Sync code:** `console.log('5')` → Output: 5
5. **Call stack empty**
6. **Microtask:** `console.log('3')` → Output: 3
7. **Microtask:** Second `.then()` executes → `console.log('4')` → Output: 4
8. **All microtasks done**
9. **Task:** `setTimeout` callback → `console.log('2')` → Output: 2

**Key insight:** Microtasks (Promises) execute before tasks (setTimeout), even if setTimeout was queued first!

---

### Q4: How do you prevent blocking the main thread?

**Answer:**

**Problem:** Long-running synchronous code freezes UI

**Solutions:**

**1. Chunk work and yield:**

```javascript
async function processLargeArray(items) {
  for (let i = 0; i < items.length; i += 100) {
    const chunk = items.slice(i, i + 100);
    processChunk(chunk);

    // Yield to event loop
    await new Promise((resolve) => setTimeout(resolve, 0));
  }
}
```

**2. Web Workers:**

```javascript
// main.js
const worker = new Worker("worker.js");
worker.postMessage({ data: largeArray });
worker.onmessage = (e) => {
  console.log("Result:", e.data);
};

// worker.js
self.onmessage = (e) => {
  const result = expensiveComputation(e.data);
  self.postMessage(result);
};
```

**3. requestIdleCallback:**

```javascript
function doWork() {
  requestIdleCallback((deadline) => {
    while (deadline.timeRemaining() > 0 && tasks.length > 0) {
      performTask(tasks.pop());
    }

    if (tasks.length > 0) {
      doWork(); // Schedule more work
    }
  });
}
```

---

### Q5: What happens in this code and why?

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}

// Output: 3, 3, 3 (not 0, 1, 2)
```

**Answer:**

**Problem:** Closure captures reference to `i`, not value

**Why:**

1. Loop executes synchronously (3 iterations)
2. `i` reaches 3
3. Call stack empties
4. Three setTimeout callbacks in task queue
5. Each callback executes, logging `i` (which is 3)

**Solutions:**

**1. Use `let` (block scope):**

```javascript
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// Output: 0, 1, 2
// Each iteration has its own `i`
```

**2. Create closure with IIFE:**

```javascript
for (var i = 0; i < 3; i++) {
  (function (j) {
    setTimeout(() => console.log(j), 0);
  })(i);
}
// Output: 0, 1, 2
```

**3. Pass value to setTimeout:**

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(console.log, 0, i);
}
// Output: 0, 1, 2
```

---

## 🧩 11. Design Patterns

### Pattern 1: Scheduler

```javascript
class TaskScheduler {
  constructor() {
    this.queue = [];
    this.running = false;
  }

  schedule(task, priority = "normal") {
    const item = { task, priority };

    if (priority === "high") {
      queueMicrotask(() => task());
    } else if (priority === "normal") {
      this.queue.push(item);
      this.run();
    } else {
      requestIdleCallback(() => task());
    }
  }

  async run() {
    if (this.running) return;
    this.running = true;

    while (this.queue.length > 0) {
      const { task } = this.queue.shift();
      await task();

      // Yield every 5 tasks
      if (this.queue.length % 5 === 0) {
        await new Promise((r) => setTimeout(r, 0));
      }
    }

    this.running = false;
  }
}
```

---

## 🎯 Summary

The event loop is JavaScript's heartbeat:

- **Single-threaded** but handles async via queues
- **Microtasks** (Promises) > **Tasks** (setTimeout)
- **Renders** happen between tasks
- **Long tasks** block UI → chunk work

**Production checklist:**

- ✅ Understand microtask vs task priority
- ✅ Yield to main thread for long operations
- ✅ Use requestAnimationFrame for animations
- ✅ Handle async errors properly
- ✅ Avoid microtask starvation
- ✅ Profile with Performance API

Master the event loop to build responsive applications! ⚡
