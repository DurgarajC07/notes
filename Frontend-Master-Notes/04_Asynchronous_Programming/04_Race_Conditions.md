# Race Conditions

> **Handling async race conditions, debouncing, and throttling**

---

## 🎯 Overview

Race conditions occur when the timing or order of events affects program correctness. Understanding and preventing race conditions is crucial for building reliable async applications.

---

## 📚 Core Concepts

### **Concept 1: What Are Race Conditions?**

A race condition occurs when multiple async operations compete, and the outcome depends on their timing:

```javascript
// ❌ Race condition
let data = null;

fetchData().then((result) => {
  data = result; // Which arrives first?
});

fetchData().then((result) => {
  data = result; // This might overwrite the first
});

// data could be from either call
```

**Real-world example:**

```javascript
// ❌ User types fast, results arrive out of order
function searchAsUserTypes(query) {
  fetch(`/api/search?q=${query}`)
    .then((r) => r.json())
    .then((results) => {
      displayResults(results); // Wrong results if responses are out of order
    });
}

// User types: 'a', 'ab', 'abc'
// Requests: 'a' (slow), 'ab' (fast), 'abc' (medium)
// Responses: 'ab' arrives, 'abc' arrives, 'a' arrives (wrong!)
```

### **Concept 2: Debouncing**

Debounce delays execution until after a period of inactivity:

```javascript
// Debounce implementation
function debounce(func, delay) {
  let timeoutId;

  return function (...args) {
    clearTimeout(timeoutId);

    timeoutId = setTimeout(() => {
      func.apply(this, args);
    }, delay);
  };
}

// Usage: search only after user stops typing
const search = debounce((query) => {
  fetch(`/api/search?q=${query}`)
    .then((r) => r.json())
    .then((results) => displayResults(results));
}, 300); // 300ms delay

input.addEventListener("input", (e) => {
  search(e.target.value);
});
```

**With leading edge:**

```javascript
function debounce(func, delay, immediate = false) {
  let timeoutId;

  return function (...args) {
    const callNow = immediate && !timeoutId;

    clearTimeout(timeoutId);

    timeoutId = setTimeout(() => {
      timeoutId = null;
      if (!immediate) {
        func.apply(this, args);
      }
    }, delay);

    if (callNow) {
      func.apply(this, args);
    }
  };
}
```

### **Concept 3: Throttling**

Throttle limits execution to once per time period:

```javascript
// Throttle implementation
function throttle(func, limit) {
  let inThrottle;

  return function (...args) {
    if (!inThrottle) {
      func.apply(this, args);
      inThrottle = true;

      setTimeout(() => {
        inThrottle = false;
      }, limit);
    }
  };
}

// Usage: limit scroll event handling
const handleScroll = throttle(() => {
  console.log("Scroll position:", window.scrollY);
}, 100); // Max once per 100ms

window.addEventListener("scroll", handleScroll);
```

**Advanced throttle with trailing edge:**

```javascript
function throttle(func, limit) {
  let inThrottle;
  let lastArgs;

  return function (...args) {
    if (!inThrottle) {
      func.apply(this, args);
      inThrottle = true;

      setTimeout(() => {
        inThrottle = false;
        if (lastArgs) {
          func.apply(this, lastArgs);
          lastArgs = null;
        }
      }, limit);
    } else {
      lastArgs = args;
    }
  };
}
```

---

## 🔥 Practical Examples

### **Example 1: Request Cancellation**

Cancel outdated requests to prevent race conditions:

```javascript
class SearchController {
  constructor() {
    this.abortController = null;
  }

  async search(query) {
    // Cancel previous request
    if (this.abortController) {
      this.abortController.abort();
    }

    this.abortController = new AbortController();

    try {
      const response = await fetch(`/api/search?q=${query}`, {
        signal: this.abortController.signal,
      });

      const results = await response.json();
      this.displayResults(results);
    } catch (error) {
      if (error.name === "AbortError") {
        console.log("Request cancelled");
      } else {
        console.error(error);
      }
    }
  }

  displayResults(results) {
    console.log("Results:", results);
  }
}

const controller = new SearchController();
input.addEventListener("input", (e) => {
  controller.search(e.target.value);
});
```

### **Example 2: Request ID Tracking**

```javascript
class AsyncOperationTracker {
  constructor() {
    this.latestRequestId = 0;
  }

  async fetchData(query) {
    const requestId = ++this.latestRequestId;

    const response = await fetch(`/api/data?q=${query}`);
    const data = await response.json();

    // Only process if this is still the latest request
    if (requestId === this.latestRequestId) {
      this.displayData(data);
    } else {
      console.log("Ignoring outdated response");
    }
  }

  displayData(data) {
    console.log("Data:", data);
  }
}
```

### **Example 3: Debounced Autocomplete**

```javascript
class Autocomplete {
  constructor(input, options = {}) {
    this.input = input;
    this.delay = options.delay || 300;
    this.minLength = options.minLength || 2;
    this.abortController = null;

    this.debouncedSearch = this.debounce(this.search.bind(this), this.delay);

    this.input.addEventListener("input", (e) => {
      this.debouncedSearch(e.target.value);
    });
  }

  debounce(func, delay) {
    let timeoutId;
    return function (...args) {
      clearTimeout(timeoutId);
      timeoutId = setTimeout(() => func(...args), delay);
    };
  }

  async search(query) {
    if (query.length < this.minLength) {
      this.clearResults();
      return;
    }

    if (this.abortController) {
      this.abortController.abort();
    }

    this.abortController = new AbortController();

    try {
      const response = await fetch(`/api/autocomplete?q=${query}`, {
        signal: this.abortController.signal,
      });

      const results = await response.json();
      this.displayResults(results);
    } catch (error) {
      if (error.name !== "AbortError") {
        console.error(error);
      }
    }
  }

  displayResults(results) {
    console.log("Autocomplete results:", results);
  }

  clearResults() {
    console.log("Clearing results");
  }
}

const autocomplete = new Autocomplete(document.querySelector("#search"), {
  delay: 300,
  minLength: 2,
});
```

### **Example 4: Infinite Scroll with Throttle**

```javascript
class InfiniteScroll {
  constructor(options = {}) {
    this.loading = false;
    this.threshold = options.threshold || 100;

    this.handleScroll = this.throttle(this.checkScroll.bind(this), 200);

    window.addEventListener("scroll", this.handleScroll);
  }

  throttle(func, limit) {
    let inThrottle;
    return function (...args) {
      if (!inThrottle) {
        func.apply(this, args);
        inThrottle = true;
        setTimeout(() => (inThrottle = false), limit);
      }
    };
  }

  async checkScroll() {
    const scrollPosition = window.scrollY + window.innerHeight;
    const pageHeight = document.documentElement.scrollHeight;

    if (scrollPosition >= pageHeight - this.threshold && !this.loading) {
      await this.loadMore();
    }
  }

  async loadMore() {
    this.loading = true;

    try {
      const response = await fetch("/api/items?page=" + this.currentPage);
      const items = await response.json();
      this.appendItems(items);
      this.currentPage++;
    } finally {
      this.loading = false;
    }
  }

  appendItems(items) {
    console.log("Appending items:", items);
  }
}
```

---

## 🏗️ Best Practices

1. **Use AbortController for fetch** - Cancel outdated requests

   ```javascript
   const controller = new AbortController();
   fetch(url, { signal: controller.signal });
   controller.abort(); // Cancel
   ```

2. **Debounce user input** - Wait for user to stop typing

   ```javascript
   const debouncedSearch = debounce(search, 300);
   ```

3. **Throttle frequent events** - Limit scroll, resize, mousemove handlers

   ```javascript
   const throttledHandler = throttle(handler, 100);
   ```

4. **Track request IDs** - Ignore outdated responses

   ```javascript
   const requestId = ++this.latestRequestId;
   if (requestId === this.latestRequestId) {
     /* process */
   }
   ```

5. **Use flags to prevent duplicate operations** - Like preventing double-submit
   ```javascript
   if (this.loading) return;
   this.loading = true;
   ```

---

## 🧪 Interview Questions

### **Q1: What's the difference between debounce and throttle?**

**Answer:**

- **Debounce** delays execution until after inactivity. Use for search-as-you-type.
  - Example: Fire after user stops typing for 300ms
- **Throttle** limits execution to once per period. Use for scroll/resize handlers.
  - Example: Fire at most once every 100ms

### **Q2: How do you prevent race conditions in async operations?**

**Answer:**

1. **Cancel outdated requests** using AbortController
2. **Track request IDs** and ignore old responses
3. **Use flags** to prevent concurrent operations
4. **Debounce** to reduce number of requests
5. **Serialize operations** using async/await

---

## 📚 Key Takeaways

- Race conditions occur when async operation timing affects correctness
- Debounce delays execution until after inactivity period
- Throttle limits execution to once per time period
- Use AbortController to cancel outdated fetch requests
- Track request IDs to ignore stale responses
- Debounce user input, throttle frequent events like scroll
- Always handle cancellation errors gracefully

---

**Master race conditions for production-ready code!**
