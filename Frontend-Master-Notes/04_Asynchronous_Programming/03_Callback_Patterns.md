# Callback Patterns

> **Error-first callbacks, callback hell, and mitigation strategies**

---

## 🎯 Overview

Callbacks are functions passed as arguments to be executed later. While largely replaced by Promises and async/await, understanding callback patterns is essential for working with legacy code and Node.js APIs.

---

## 📚 Core Concepts

### **Concept 1: Error-First Callbacks (Node.js Convention)**

Node.js uses error-first callback pattern:

```javascript
// Pattern: callback(error, result)
fs.readFile("file.txt", "utf8", (error, data) => {
  if (error) {
    console.error("Error:", error);
    return;
  }
  console.log("Data:", data);
});

// Implementing error-first callbacks
function fetchData(url, callback) {
  // Simulate async operation
  setTimeout(() => {
    const error = null; // or new Error('Failed')
    const data = { id: 1, name: "Alice" };

    callback(error, data);
  }, 1000);
}

// Usage
fetchData("/api/user", (error, data) => {
  if (error) {
    console.error(error);
    return;
  }
  console.log(data);
});
```

### **Concept 2: Callback Hell (Pyramid of Doom)**

Nested callbacks become hard to read and maintain:

```javascript
// ❌ Callback hell
getUser(userId, (error, user) => {
  if (error) {
    console.error(error);
    return;
  }

  getPosts(user.id, (error, posts) => {
    if (error) {
      console.error(error);
      return;
    }

    getComments(posts[0].id, (error, comments) => {
      if (error) {
        console.error(error);
        return;
      }

      displayData(user, posts, comments);
    });
  });
});
```

### **Concept 3: Callback Mitigation Strategies**

**Strategy 1: Named Functions**

```javascript
// ✅ Better - named functions
function handleUser(error, user) {
  if (error) return handleError(error);
  getPosts(user.id, handlePosts);
}

function handlePosts(error, posts) {
  if (error) return handleError(error);
  getComments(posts[0].id, handleComments);
}

function handleComments(error, comments) {
  if (error) return handleError(error);
  displayData(comments);
}

function handleError(error) {
  console.error("Error:", error);
}

getUser(userId, handleUser);
```

**Strategy 2: Promisify**

```javascript
// Convert callbacks to Promises
function promisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (error, result) => {
        if (error) reject(error);
        else resolve(result);
      });
    });
  };
}

// Usage
const getUserPromise = promisify(getUser);
const getPostsPromise = promisify(getPosts);

// Clean async/await
async function fetchData() {
  try {
    const user = await getUserPromise(userId);
    const posts = await getPostsPromise(user.id);
    return { user, posts };
  } catch (error) {
    console.error(error);
  }
}
```

---

## 🔥 Practical Examples

### **Example 1: Async Control Flow Library**

```javascript
// Simple async series implementation
function series(tasks, callback) {
  let currentIndex = 0;
  const results = [];

  function next(error, result) {
    if (error) {
      return callback(error);
    }

    if (currentIndex > 0) {
      results.push(result);
    }

    if (currentIndex >= tasks.length) {
      return callback(null, results);
    }

    const task = tasks[currentIndex++];
    task(next);
  }

  next();
}

// Usage
series(
  [
    (cb) => setTimeout(() => cb(null, "Task 1"), 100),
    (cb) => setTimeout(() => cb(null, "Task 2"), 100),
    (cb) => setTimeout(() => cb(null, "Task 3"), 100),
  ],
  (error, results) => {
    console.log(results); // ['Task 1', 'Task 2', 'Task 3']
  },
);
```

### **Example 2: Callback Timeout**

```javascript
function withTimeout(fn, timeout, callback) {
  let called = false;

  const timer = setTimeout(() => {
    if (!called) {
      called = true;
      callback(new Error("Timeout"));
    }
  }, timeout);

  fn((error, result) => {
    if (!called) {
      called = true;
      clearTimeout(timer);
      callback(error, result);
    }
  });
}

// Usage
withTimeout(
  (cb) => fetchData(url, cb),
  5000,
  (error, data) => {
    if (error) {
      console.error("Failed or timed out:", error);
    } else {
      console.log("Success:", data);
    }
  },
);
```

### **Example 3: Parallel Execution**

```javascript
function parallel(tasks, callback) {
  let completed = 0;
  const results = [];
  let hasError = false;

  if (tasks.length === 0) {
    return callback(null, results);
  }

  tasks.forEach((task, index) => {
    task((error, result) => {
      if (hasError) return;

      if (error) {
        hasError = true;
        return callback(error);
      }

      results[index] = result;
      completed++;

      if (completed === tasks.length) {
        callback(null, results);
      }
    });
  });
}

// Usage
parallel(
  [
    (cb) => fetchUser(1, cb),
    (cb) => fetchPosts(1, cb),
    (cb) => fetchComments(1, cb),
  ],
  (error, [user, posts, comments]) => {
    if (error) {
      console.error(error);
    } else {
      console.log({ user, posts, comments });
    }
  },
);
```

---

## 🏗️ Best Practices

1. **Follow error-first convention** - First parameter is error, second is result

   ```javascript
   callback(error, result);
   ```

2. **Always check for errors first** - Handle errors before processing results

   ```javascript
   if (error) return handleError(error);
   ```

3. **Call callback exactly once** - Prevent multiple invocations

   ```javascript
   let called = false;
   if (called) return;
   called = true;
   callback(error, result);
   ```

4. **Use named functions** - Avoid deep nesting

   ```javascript
   fetchData(url, handleData);
   ```

5. **Prefer Promises/async-await** - For new code
   ```javascript
   const data = await fetchData(url);
   ```

---

## 🧪 Interview Questions

### **Q1: What is the error-first callback pattern?**

**Answer:** Error-first callbacks are a Node.js convention where the first parameter is an error (or null if no error), and subsequent parameters are results:

```javascript
fs.readFile("file.txt", (error, data) => {
  if (error) {
    /* handle error */
  }
  // use data
});
```

This pattern standardizes error handling across async operations.

### **Q2: What is callback hell and how do you solve it?**

**Answer:** Callback hell (pyramid of doom) is deeply nested callbacks that are hard to read:

```javascript
a(() => b(() => c(() => d())));
```

**Solutions:**

1. Named functions to flatten structure
2. Promisify callbacks and use async/await
3. Use async libraries (async.js)
4. Prefer Promises for new code

---

## 📚 Key Takeaways

- Error-first callbacks: first parameter is error, second is result
- Callback hell occurs with deep nesting - use named functions or Promises
- Always check errors first before processing results
- Call callbacks exactly once to avoid bugs
- Prefer Promises and async/await for new code
- Promisify callback-based APIs for cleaner code

---

**Master callback patterns for production-ready code!**
