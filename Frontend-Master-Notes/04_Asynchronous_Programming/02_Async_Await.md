# Async/Await

> **Modern async patterns with syntactic sugar over Promises**

---

## 🎯 Overview

Async/await provides synchronous-looking syntax for asynchronous code, making it easier to read and maintain. It's built on Promises and is the preferred way to handle async operations in modern JavaScript.

---

## 📚 Core Concepts

### **Concept 1: Async Function Basics**

`async` functions always return a Promise:

```javascript
// Async function declaration
async function fetchUser() {
  return { id: 1, name: "Alice" };
}

// Returns Promise<{ id: number, name: string }>
fetchUser().then((user) => console.log(user));

// Async arrow function
const fetchUser = async () => {
  return { id: 1, name: "Alice" };
};

// Async method
class API {
  async getUser() {
    return { id: 1, name: "Alice" };
  }
}
```

### **Concept 2: Await Keyword**

`await` pauses execution until Promise resolves:

```javascript
// Without await - Promise
function getUser() {
  return fetch("/api/user").then((r) => r.json());
}

// With await - cleaner syntax
async function getUser() {
  const response = await fetch("/api/user");
  const user = await response.json();
  return user;
}

// Sequential operations
async function fetchData() {
  const user = await fetchUser(); // Wait
  const posts = await fetchPosts(user.id); // Then wait
  const comments = await fetchComments(posts[0].id); // Then wait
  return { user, posts, comments };
}
```

### **Concept 3: Error Handling**

Use try/catch with async/await:

```javascript
// Try/catch for errors
async function fetchUser() {
  try {
    const response = await fetch("/api/user");

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    const user = await response.json();
    return user;
  } catch (error) {
    console.error("Failed to fetch user:", error);
    throw error; // Re-throw or return default
  }
}

// Multiple try/catch blocks
async function processData() {
  let user;
  try {
    user = await fetchUser();
  } catch (error) {
    console.error("User fetch failed:", error);
    return null;
  }

  try {
    const posts = await fetchPosts(user.id);
    return posts;
  } catch (error) {
    console.error("Posts fetch failed:", error);
    return [];
  }
}
```

---

## 🔥 Practical Examples

### **Example 1: Sequential vs Parallel**

```javascript
// ❌ Sequential - slow (3 seconds total)
async function fetchDataSequential() {
  const user = await fetchUser(); // 1 second
  const posts = await fetchPosts(); // 1 second
  const comments = await fetchComments(); // 1 second
  return { user, posts, comments };
}

// ✅ Parallel - fast (1 second total)
async function fetchDataParallel() {
  const [user, posts, comments] = await Promise.all([
    fetchUser(),
    fetchPosts(),
    fetchComments(),
  ]);
  return { user, posts, comments };
}

// ✅ Dependent parallel - optimal
async function fetchDataOptimal() {
  const user = await fetchUser();

  // These can run in parallel
  const [posts, friends] = await Promise.all([
    fetchPosts(user.id),
    fetchFriends(user.id),
  ]);

  return { user, posts, friends };
}
```

### **Example 2: Async Loops**

```javascript
// ❌ Bad - doesn't wait
async function processItems(items) {
  items.forEach(async (item) => {
    await processItem(item); // Doesn't wait!
  });
  console.log("Done"); // Runs before processing
}

// ✅ Sequential processing
async function processItemsSequential(items) {
  for (const item of items) {
    await processItem(item);
  }
  console.log("Done"); // Runs after all processing
}

// ✅ Parallel processing
async function processItemsParallel(items) {
  await Promise.all(items.map((item) => processItem(item)));
  console.log("Done");
}

// ✅ Parallel with results
async function processItemsWithResults(items) {
  const results = await Promise.all(
    items.map(async (item) => {
      const processed = await processItem(item);
      return { item, processed };
    }),
  );
  return results;
}
```

### **Example 3: Error Handling Patterns**

```javascript
// Pattern 1: Wrapper function
async function safeAsync(promise) {
  try {
    const data = await promise;
    return [null, data];
  } catch (error) {
    return [error, null];
  }
}

// Usage
const [error, user] = await safeAsync(fetchUser());
if (error) {
  console.error(error);
} else {
  console.log(user);
}

// Pattern 2: Default values
async function fetchUserSafe(id) {
  try {
    return await fetchUser(id);
  } catch (error) {
    console.error(error);
    return { id, name: "Unknown", email: null };
  }
}

// Pattern 3: Retry logic
async function fetchWithRetry(fn, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === retries - 1) throw error;
      await new Promise((r) => setTimeout(r, 1000 * (i + 1)));
    }
  }
}
```

### **Example 4: Async IIFE**

```javascript
// Top-level await (ES2022+)
const user = await fetchUser();

// Before ES2022 - use IIFE
(async () => {
  const user = await fetchUser();
  console.log(user);
})();

// With error handling
(async () => {
  try {
    const user = await fetchUser();
    console.log(user);
  } catch (error) {
    console.error(error);
  }
})();
```

---

## 🏗️ Best Practices

1. **Use await Promise.all() for independent operations** - Don't wait unnecessarily

   ```javascript
   const [a, b] = await Promise.all([fetchA(), fetchB()]);
   ```

2. **Always use try/catch** - Handle errors explicitly

   ```javascript
   try {
     await riskyOperation();
   } catch (error) {
     handleError(error);
   }
   ```

3. **Be careful with async in loops** - Understand sequential vs parallel

   ```javascript
   // Sequential
   for (const item of items) await process(item);
   // Parallel
   await Promise.all(items.map(process));
   ```

4. **Don't mix async/await with .then()** - Choose one style

   ```javascript
   // ❌ Mixed
   const data = await fetch(url).then((r) => r.json());
   // ✅ Consistent
   const response = await fetch(url);
   const data = await response.json();
   ```

5. **Return early** - Avoid nested try/catch
   ```javascript
   async function process() {
     const user = await fetchUser();
     if (!user) return null;

     const posts = await fetchPosts(user.id);
     return posts;
   }
   ```

---

## 🧪 Interview Questions

### **Q1: What's the difference between async/await and Promises?**

**Answer:** Async/await is syntactic sugar over Promises. `async` functions return Promises, and `await` pauses execution until a Promise resolves. They make async code look synchronous and are easier to read.

```javascript
// Promise
fetchUser().then(user => fetchPosts(user.id)).then(posts => ...);

// Async/await
const user = await fetchUser();
const posts = await fetchPosts(user.id);
```

### **Q2: Can you use await outside async functions?**

**Answer:** Before ES2022, no. You needed an async function or IIFE. ES2022+ supports **top-level await** in modules:

```javascript
// ES2022+ (modules only)
const data = await fetch("/api/data");

// Before ES2022
(async () => {
  const data = await fetch("/api/data");
})();
```

### **Q3: How do you handle multiple async operations efficiently?**

**Answer:** Use `Promise.all()` for independent operations:

```javascript
// ❌ Sequential - slow
const a = await fetchA();
const b = await fetchB();

// ✅ Parallel - fast
const [a, b] = await Promise.all([fetchA(), fetchB()]);
```

---

## 📚 Key Takeaways

- Async/await is syntactic sugar over Promises, making async code look synchronous
- `async` functions always return Promises
- `await` pauses execution until Promise resolves
- Use try/catch for error handling with async/await
- Use Promise.all() for parallel operations to avoid sequential delays
- Be careful with async in loops - understand when operations run
- Top-level await is available in ES2022+ modules

---

**Master async/await for production-ready code!**
