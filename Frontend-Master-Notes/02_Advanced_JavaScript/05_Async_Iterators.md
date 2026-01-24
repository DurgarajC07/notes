# Async Iterators & Generators

> **Asynchronous iteration with for-await-of and async generators**

---

## 🎯 What are Async Iterators?

**Async Iterators** allow you to iterate over asynchronous data sources (like streams, paginated APIs, or real-time data) using a familiar iteration syntax.

```javascript
// Async Iterator Protocol
const asyncIterator = {
  async next() {
    // Returns a promise that resolves to { value, done }
    return { value: await fetchData(), done: false };
  },
};
```

---

## 📚 Async Iterable Protocol

An object is **async iterable** if it implements `Symbol.asyncIterator`:

```javascript
const asyncIterable = {
  [Symbol.asyncIterator]() {
    let i = 0;
    return {
      async next() {
        if (i < 3) {
          await new Promise((resolve) => setTimeout(resolve, 1000));
          return { value: i++, done: false };
        }
        return { done: true };
      },
    };
  },
};

// Using for-await-of
(async () => {
  for await (const value of asyncIterable) {
    console.log(value); // 0, 1, 2 (with 1 second delay each)
  }
})();
```

---

## 🔥 Async Generator Functions

**Async generators** combine generators with async/await:

```javascript
async function* asyncGenerator() {
  yield await Promise.resolve(1);
  yield await Promise.resolve(2);
  yield await Promise.resolve(3);
}

(async () => {
  for await (const value of asyncGenerator()) {
    console.log(value); // 1, 2, 3
  }
})();
```

### **With Delays**

```javascript
async function* countdown(from) {
  for (let i = from; i > 0; i--) {
    await new Promise((resolve) => setTimeout(resolve, 1000));
    yield i;
  }
}
```

---

## 🌐 Real-World Use Cases

### **1. Paginated API Fetching**

```javascript
async function* fetchPages(apiUrl) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(`${apiUrl}?page=${page}`);
    const data = await response.json();

    yield data.items;

    hasMore = data.hasMore;
    page++;
  }
}

// Usage
(async () => {
  for await (const items of fetchPages("https://api.example.com/data")) {
    console.log("Processing page:", items);
  }
})();
```

### **2. Real-Time Data Streams**

```javascript
async function* watchStock(symbol) {
  const ws = new WebSocket(`wss://api.example.com/stocks/${symbol}`);

  const queue = [];
  let resolveNext;

  ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    if (resolveNext) {
      resolveNext(data);
      resolveNext = null;
    } else {
      queue.push(data);
    }
  };

  while (true) {
    if (queue.length > 0) {
      yield queue.shift();
    } else {
      yield await new Promise((resolve) => (resolveNext = resolve));
    }
  }
}
```

---

## 🎯 Best Practices

1. **Use async generators** instead of manually implementing async iterators
2. **Handle cleanup** in finally blocks for resources
3. **Implement cancellation** when needed
4. **Use `for-await-of`** instead of manual iteration
5. **Handle errors gracefully** with try-catch
6. **Be mindful of memory** with infinite iterators

---

## 🧪 Interview Questions

### **Q1: What's the difference between regular and async iterators?**

**Answer:**

- **Regular Iterator**: `next()` returns `{value, done}` synchronously
- **Async Iterator**: `next()` returns a Promise that resolves to `{value, done}`

### **Q2: Can you use `break` in `for-await-of`?**

**Answer:** Yes, `break`, `continue`, and `return` all work in `for-await-of` loops.

---

## 📚 Key Takeaways

- **Async iterators** enable iteration over async data sources
- **`for-await-of`** is the standard way to consume async iterables
- **Async generators** (`async function*`) simplify async iterator creation
- **Perfect for**: API pagination, streams, real-time data, file processing

---

**Master async iteration for elegant async data processing!**
