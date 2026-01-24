# Parallel Execution

> **Promise.all, Promise.race, Promise.allSettled, and Promise.any**

---

## 🎯 Overview

Parallel execution allows multiple async operations to run simultaneously, improving performance. JavaScript provides several Promise combinators for different parallel execution patterns.

---

## 📚 Core Concepts

### **Concept 1: Promise.all()**

Runs all promises in parallel, waits for all to complete:

```javascript
// Promise.all - all must succeed
const promises = [
  fetch("/api/users"),
  fetch("/api/posts"),
  fetch("/api/comments"),
];

Promise.all(promises)
  .then((responses) => Promise.all(responses.map((r) => r.json())))
  .then(([users, posts, comments]) => {
    console.log({ users, posts, comments });
  })
  .catch((error) => {
    // If ANY promise rejects, entire operation fails
    console.error("One or more requests failed:", error);
  });

// With async/await
try {
  const [users, posts, comments] = await Promise.all([
    fetchUsers(),
    fetchPosts(),
    fetchComments(),
  ]);
  console.log({ users, posts, comments });
} catch (error) {
  console.error("Failed:", error);
}
```

**Key characteristics:**

- All promises run in parallel
- Resolves when ALL promises resolve
- Rejects immediately if ANY promise rejects
- Returns array of results in same order

### **Concept 2: Promise.race()**

Resolves/rejects as soon as the first promise settles:

```javascript
// Promise.race - first to finish wins
const timeout = new Promise((_, reject) =>
  setTimeout(() => reject(new Error("Timeout")), 5000),
);

const dataFetch = fetch("/api/data");

Promise.race([dataFetch, timeout])
  .then((response) => response.json())
  .then((data) => console.log("Got data:", data))
  .catch((error) => console.error("Failed or timed out:", error));

// First promise to settle (resolve or reject) determines outcome
```

**Use cases:**

- Implementing timeouts
- Racing multiple sources for fastest response
- Cancellation patterns

### **Concept 3: Promise.allSettled()**

Waits for all promises to settle (resolve or reject), never rejects:

```javascript
// Promise.allSettled - wait for all, handle partial failures
const promises = [
  fetch("/api/users"),
  fetch("/api/posts"), // This might fail
  fetch("/api/comments"),
];

const results = await Promise.allSettled(promises);

// Process results
results.forEach((result, index) => {
  if (result.status === "fulfilled") {
    console.log(`Promise ${index} succeeded:`, result.value);
  } else {
    console.error(`Promise ${index} failed:`, result.reason);
  }
});

// Example output:
/*
[
  { status: 'fulfilled', value: Response },
  { status: 'rejected', reason: Error },
  { status: 'fulfilled', value: Response }
]
*/
```

**Key characteristics:**

- Waits for ALL promises to settle
- Never rejects
- Returns array of result objects with `status` and `value`/`reason`

### **Concept 4: Promise.any()**

Resolves as soon as ANY promise resolves (opposite of Promise.all):

```javascript
// Promise.any - first success wins
const mirrors = [
  fetch("https://mirror1.example.com/data"),
  fetch("https://mirror2.example.com/data"),
  fetch("https://mirror3.example.com/data"),
];

Promise.any(mirrors)
  .then((response) => response.json())
  .then((data) => console.log("Got data from fastest mirror:", data))
  .catch((error) => {
    // AggregateError - all promises rejected
    console.error("All mirrors failed:", error.errors);
  });
```

**Key characteristics:**

- Resolves with first successful promise
- Rejects only if ALL promises reject (AggregateError)
- Ignores rejections until all fail

---

## 🔥 Practical Examples

### **Example 1: Concurrent API Calls with Fallback**

```javascript
async function fetchDataWithFallback() {
  try {
    const [users, posts] = await Promise.all([fetchUsers(), fetchPosts()]);
    return { users, posts };
  } catch (error) {
    console.error("Primary fetch failed, using cache");

    // Fallback: load from cache
    const [users, posts] = await Promise.all([
      getCachedUsers(),
      getCachedPosts(),
    ]);
    return { users, posts };
  }
}
```

### **Example 2: Request with Timeout**

```javascript
function fetchWithTimeout(url, timeout = 5000) {
  return Promise.race([
    fetch(url),
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error("Request timeout")), timeout),
    ),
  ]);
}

// Usage
try {
  const response = await fetchWithTimeout("/api/data", 3000);
  const data = await response.json();
  console.log(data);
} catch (error) {
  console.error(error.message); // 'Request timeout' if slow
}
```

### **Example 3: Batch Processing with Partial Failures**

```javascript
async function batchProcess(items) {
  const promises = items.map((item) =>
    processItem(item).catch((error) => ({ error, item })),
  );

  const results = await Promise.all(promises);

  // Separate successes and failures
  const successes = results.filter((r) => !r.error);
  const failures = results.filter((r) => r.error);

  console.log(
    `Processed: ${successes.length} succeeded, ${failures.length} failed`,
  );

  return { successes, failures };
}

// Or using allSettled
async function batchProcessWithAllSettled(items) {
  const promises = items.map((item) => processItem(item));
  const results = await Promise.allSettled(promises);

  const successes = results
    .filter((r) => r.status === "fulfilled")
    .map((r) => r.value);

  const failures = results
    .filter((r) => r.status === "rejected")
    .map((r) => r.reason);

  return { successes, failures };
}
```

### **Example 4: Fast Mirror Selection**

```javascript
class MirrorFetcher {
  constructor(mirrors) {
    this.mirrors = mirrors;
  }

  async fetch(path) {
    // Try all mirrors, use fastest successful response
    const promises = this.mirrors.map((mirror) =>
      fetch(`${mirror}${path}`).then((r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r;
      }),
    );

    try {
      const response = await Promise.any(promises);
      return await response.json();
    } catch (error) {
      // All mirrors failed
      throw new Error("All mirrors unavailable");
    }
  }
}

const fetcher = new MirrorFetcher([
  "https://cdn1.example.com",
  "https://cdn2.example.com",
  "https://cdn3.example.com",
]);

const data = await fetcher.fetch("/api/data");
```

### **Example 5: Controlled Concurrency**

```javascript
// Process items with max concurrency
async function processWithConcurrency(items, concurrency, processor) {
  const results = [];
  const queue = [...items];
  const inProgress = [];

  while (queue.length > 0 || inProgress.length > 0) {
    // Fill up to concurrency limit
    while (inProgress.length < concurrency && queue.length > 0) {
      const item = queue.shift();
      const promise = processor(item)
        .then((result) => ({ result, item }))
        .catch((error) => ({ error, item }));

      inProgress.push(promise);
    }

    // Wait for at least one to complete
    const completed = await Promise.race(inProgress);
    results.push(completed);

    // Remove completed from inProgress
    const index = inProgress.findIndex((p) => p === completed);
    inProgress.splice(index, 1);
  }

  return results;
}

// Usage: Process 1000 items, max 10 concurrent
const results = await processWithConcurrency(
  items,
  10,
  async (item) => await processItem(item),
);
```

---

## 🏗️ Best Practices

1. **Use Promise.all for independent operations** - Runs in parallel

   ```javascript
   const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);
   ```

2. **Use Promise.allSettled for partial failures** - When some can fail

   ```javascript
   const results = await Promise.allSettled(promises);
   ```

3. **Use Promise.race for timeouts** - Add timeout to any operation

   ```javascript
   Promise.race([operation(), timeout(5000)]);
   ```

4. **Use Promise.any for fastest success** - Multiple mirrors/sources

   ```javascript
   const data = await Promise.any([mirror1(), mirror2(), mirror3()]);
   ```

5. **Limit concurrency for large batches** - Prevent resource exhaustion
   ```javascript
   await processInBatches(items, 10);
   ```

---

## 🧪 Interview Questions

### **Q1: Explain the difference between Promise.all and Promise.allSettled.**

**Answer:**

- **Promise.all()** rejects immediately if ANY promise rejects. Use when all operations must succeed.
- **Promise.allSettled()** waits for ALL promises to settle (resolve or reject), never rejects. Use when you need results from all operations regardless of success/failure.

### **Q2: When would you use Promise.race vs Promise.any?**

**Answer:**

- **Promise.race()** resolves/rejects with the FIRST settled promise (success or failure). Use for timeouts or cancellation.
- **Promise.any()** resolves with the FIRST successful promise, ignores rejections until all fail. Use for fastest successful response from multiple sources.

### **Q3: How do you implement a timeout for a Promise?**

**Answer:**

```javascript
function timeout(ms) {
  return new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Timeout")), ms),
  );
}

Promise.race([operation(), timeout(5000)]);
```

---

## 📚 Key Takeaways

- Promise.all runs promises in parallel, fails if any fails
- Promise.race settles with first promise to settle (resolve or reject)
- Promise.allSettled waits for all, returns all results, never rejects
- Promise.any resolves with first success, rejects only if all fail
- Use Promise.all for performance when operations are independent
- Use Promise.allSettled for partial failure handling
- Limit concurrency to prevent resource exhaustion
- Combine Promise.race with timeout promises for timeout functionality

---

**Master parallel execution for production-ready code!**
