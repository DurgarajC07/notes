# JavaScript Promises

> **Promise states, chaining, and error handling**

---

## 🎯 Overview

Promises represent the eventual completion (or failure) of an asynchronous operation. They provide a cleaner alternative to callbacks and are fundamental to modern JavaScript async programming.

---

## 📚 Core Concepts

### **Concept 1: Promise States**

A Promise has three states:

1. **Pending** - Initial state, neither fulfilled nor rejected
2. **Fulfilled** - Operation completed successfully
3. **Rejected** - Operation failed

```javascript
// Creating a promise
const promise = new Promise((resolve, reject) => {
  const success = true;

  if (success) {
    resolve("Operation successful"); // Fulfilled
  } else {
    reject("Operation failed"); // Rejected
  }
});

// Consuming the promise
promise
  .then((result) => console.log(result)) // Fulfilled
  .catch((error) => console.error(error)) // Rejected
  .finally(() => console.log("Cleanup")); // Always runs
```

### **Concept 2: Promise Chaining**

Chain `.then()` calls to perform sequential async operations:

```javascript
// Sequential operations
fetch("/api/user")
  .then((response) => response.json())
  .then((user) => fetch(`/api/posts/${user.id}`))
  .then((response) => response.json())
  .then((posts) => console.log(posts))
  .catch((error) => console.error(error));

// Return values are wrapped in promises
Promise.resolve(1)
  .then((x) => x + 1) // Returns Promise<2>
  .then((x) => x * 2) // Returns Promise<4>
  .then((x) => console.log(x)); // 4

// Returning a promise
Promise.resolve(1)
  .then((x) => {
    return new Promise((resolve) => {
      setTimeout(() => resolve(x * 2), 1000);
    });
  })
  .then((x) => console.log(x)); // 2 (after 1 second)
```

### **Concept 3: Error Handling**

```javascript
// Errors propagate down the chain
Promise.resolve()
  .then(() => {
    throw new Error("Oops!");
  })
  .then(() => {
    console.log("This won't run");
  })
  .catch((error) => {
    console.error("Caught:", error.message); // Caught: Oops!
  });

// Recover from errors
Promise.reject("Error")
  .catch((error) => {
    console.error(error);
    return "Recovered"; // Chain continues
  })
  .then((result) => console.log(result)); // Recovered

// Multiple catch handlers
fetch("/api/data")
  .then((response) => {
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return response.json();
  })
  .catch((error) => {
    // Handle fetch/network errors
    console.error("Network error:", error);
    throw error; // Re-throw to propagate
  })
  .then((data) => processData(data))
  .catch((error) => {
    // Handle processing errors
    console.error("Processing error:", error);
  });
```

---

## 🔥 Practical Examples

### **Example 1: Promisifying Callbacks**

```javascript
// Old callback API
function readFileCallback(path, callback) {
  fs.readFile(path, "utf8", (err, data) => {
    if (err) callback(err);
    else callback(null, data);
  });
}

// Promisified version
function readFile(path) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, "utf8", (err, data) => {
      if (err) reject(err);
      else resolve(data);
    });
  });
}

// Usage
readFile("file.txt")
  .then((data) => console.log(data))
  .catch((error) => console.error(error));

// Using util.promisify (Node.js)
const { promisify } = require("util");
const readFile = promisify(fs.readFile);
```

### **Example 2: Retry Logic**

```javascript
function fetchWithRetry(url, options = {}, retries = 3) {
  return fetch(url, options).catch((error) => {
    if (retries > 0) {
      console.log(`Retrying... (${retries} attempts left)`);
      return fetchWithRetry(url, options, retries - 1);
    }
    throw error;
  });
}

// Usage
fetchWithRetry("/api/data", {}, 3)
  .then((response) => response.json())
  .then((data) => console.log(data))
  .catch((error) => console.error("All retries failed:", error));
```

### **Example 3: Timeout Pattern**

```javascript
function timeout(ms) {
  return new Promise((_, reject) => {
    setTimeout(() => reject(new Error("Timeout")), ms);
  });
}

function fetchWithTimeout(url, ms = 5000) {
  return Promise.race([fetch(url), timeout(ms)]);
}

// Usage
fetchWithTimeout("/api/slow-endpoint", 3000)
  .then((response) => response.json())
  .catch((error) => console.error(error.message)); // 'Timeout'
```

### **Example 4: Promise Pool (Concurrency Limit)**

```javascript
async function promisePool(tasks, limit) {
  const results = [];
  const executing = [];

  for (const task of tasks) {
    const promise = Promise.resolve().then(() => task());
    results.push(promise);

    if (limit <= tasks.length) {
      const e = promise.then(() => executing.splice(executing.indexOf(e), 1));
      executing.push(e);

      if (executing.length >= limit) {
        await Promise.race(executing);
      }
    }
  }

  return Promise.all(results);
}

// Usage: Process 100 items with max 5 concurrent
const tasks = items.map((item) => () => processItem(item));
await promisePool(tasks, 5);
```

---

## 🏗️ Best Practices

1. **Always handle rejections** - Use `.catch()` or `try/catch` with async/await

   ```javascript
   promise.catch((error) => handleError(error));
   ```

2. **Return promises in chains** - Enables proper chaining

   ```javascript
   .then(data => {
     return processData(data); // Return promise
   })
   ```

3. **Use Promise.all for parallel operations** - When operations don't depend on each other

   ```javascript
   Promise.all([fetch1(), fetch2(), fetch3()]);
   ```

4. **Avoid the Promise constructor antipattern** - Don't wrap promises unnecessarily

   ```javascript
   // ❌ Bad
   return new Promise((resolve) => {
     return fetch(url).then(resolve);
   });

   // ✅ Good
   return fetch(url);
   ```

5. **Use finally for cleanup** - Runs regardless of success/failure
   ```javascript
   showLoader();
   fetch(url)
     .then(handleData)
     .catch(handleError)
     .finally(() => hideLoader());
   ```

---

## 🧪 Interview Questions

### **Q1: Explain Promise states and transitions.**

**Answer:** A Promise starts in **pending** state. It can transition to:

- **Fulfilled** via `resolve()` (success)
- **Rejected** via `reject()` (error)

Once settled (fulfilled/rejected), state cannot change. This is called being "settled" or "resolved".

### **Q2: What's the difference between `.then()` and `.catch()`?**

**Answer:** `.then(onFulfilled, onRejected)` handles both success and failure. `.catch(onRejected)` is syntactic sugar for `.then(null, onRejected)`. Use `.catch()` for cleaner error handling as it catches errors from all previous `.then()` handlers in the chain.

### **Q3: How do you handle multiple independent async operations?**

**Answer:** Use `Promise.all()` to run operations in parallel:

```javascript
const [users, posts, comments] = await Promise.all([
  fetchUsers(),
  fetchPosts(),
  fetchComments(),
]);
```

If one fails, all fail. Use `Promise.allSettled()` to handle partial failures.

---

## 📚 Key Takeaways

- Promises have three states: pending, fulfilled, rejected
- Chain `.then()` for sequential operations, use `.catch()` for errors
- Always handle rejections to avoid unhandled promise rejections
- Use `Promise.all()` for parallel operations
- `.finally()` runs cleanup code regardless of outcome
- Promisify callback-based APIs for cleaner async code

---

**Master javascript promises for production-ready code!**
