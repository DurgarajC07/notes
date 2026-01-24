# Generators & Iterators in JavaScript

> **Master iteration protocols and generator functions for advanced control flow**

---

## 🎯 What are Iterators?

An **iterator** is an object that defines a sequence and potentially a return value upon termination. It implements the Iterator protocol by having a `next()` method that returns `{value, done}`.

---

## 📚 Iterator Protocol

### **Basic Iterator**

```javascript
const iterator = {
  current: 0,
  last: 5,

  next() {
    if (this.current <= this.last) {
      return { value: this.current++, done: false };
    } else {
      return { value: undefined, done: true };
    }
  },
};

console.log(iterator.next()); // { value: 0, done: false }
console.log(iterator.next()); // { value: 1, done: false }
// ... continues until done: true
```

### **Iterable Protocol**

An object is **iterable** if it implements the `@@iterator` method (accessible via `Symbol.iterator`):

```javascript
const iterableObject = {
  data: [1, 2, 3, 4, 5],

  [Symbol.iterator]() {
    let index = 0;
    const data = this.data;

    return {
      next() {
        if (index < data.length) {
          return { value: data[index++], done: false };
        } else {
          return { value: undefined, done: true };
        }
      },
    };
  },
};

// Can now use for...of
for (const value of iterableObject) {
  console.log(value); // 1, 2, 3, 4, 5
}

// Or spread operator
console.log([...iterableObject]); // [1, 2, 3, 4, 5]
```

---

## 🔥 Generator Functions

**Generators** are functions that can be paused and resumed, making iterator creation much simpler.

### **Basic Generator**

```javascript
function* numberGenerator() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = numberGenerator();

console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }
console.log(gen.next()); // { value: 3, done: false }
console.log(gen.next()); // { value: undefined, done: true }

// Using for...of
for (const num of numberGenerator()) {
  console.log(num); // 1, 2, 3
}
```

### **Generator with Parameters**

```javascript
function* range(start, end, step = 1) {
  for (let i = start; i <= end; i += step) {
    yield i;
  }
}

console.log([...range(1, 10, 2)]); // [1, 3, 5, 7, 9]

// Infinite generator
function* infiniteSequence() {
  let i = 0;
  while (true) {
    yield i++;
  }
}

const infinite = infiniteSequence();
console.log(infinite.next().value); // 0
console.log(infinite.next().value); // 1
console.log(infinite.next().value); // 2
```

---

## 🎨 Advanced Generator Patterns

### **1. Passing Values to Generators**

```javascript
function* generatorWithInput() {
  const x = yield "Give me X";
  const y = yield "Give me Y";
  return x + y;
}

const gen = generatorWithInput();

console.log(gen.next()); // { value: 'Give me X', done: false }
console.log(gen.next(10)); // { value: 'Give me Y', done: false }
console.log(gen.next(20)); // { value: 30, done: true }
```

### **2. Generator Delegation (yield\*)**

```javascript
function* generator1() {
  yield 1;
  yield 2;
}

function* generator2() {
  yield 3;
  yield 4;
}

function* combinedGenerator() {
  yield* generator1();
  yield* generator2();
  yield 5;
}

console.log([...combinedGenerator()]); // [1, 2, 3, 4, 5]

// Delegating to iterables
function* delegateToArray() {
  yield* [1, 2, 3];
  yield* "abc";
}

console.log([...delegateToArray()]); // [1, 2, 3, 'a', 'b', 'c']
```

### **3. Error Handling in Generators**

```javascript
function* generatorWithError() {
  try {
    yield 1;
    yield 2;
    yield 3;
  } catch (error) {
    console.log("Error caught:", error.message);
    yield "error handled";
  }
}

const gen = generatorWithError();

console.log(gen.next()); // { value: 1, done: false }
console.log(gen.throw(new Error("Something went wrong")));
// Logs: "Error caught: Something went wrong"
// Returns: { value: 'error handled', done: false }
```

### **4. Generator Return Method**

```javascript
function* generator() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = generator();

console.log(gen.next()); // { value: 1, done: false }
console.log(gen.return(99)); // { value: 99, done: true }
console.log(gen.next()); // { value: undefined, done: true }
```

---

## 🚀 Real-World Use Cases

### **1. Lazy Evaluation**

```javascript
function* fibonacci() {
  let [prev, curr] = [0, 1];

  while (true) {
    yield curr;
    [prev, curr] = [curr, prev + curr];
  }
}

// Get first 10 Fibonacci numbers
function take(n, iterable) {
  const result = [];
  const iterator = iterable[Symbol.iterator]();

  for (let i = 0; i < n; i++) {
    const { value, done } = iterator.next();
    if (done) break;
    result.push(value);
  }

  return result;
}

console.log(take(10, fibonacci()));
// [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
```

### **2. Pagination**

```javascript
function* paginate(array, pageSize) {
  for (let i = 0; i < array.length; i += pageSize) {
    yield array.slice(i, i + pageSize);
  }
}

const data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

for (const page of paginate(data, 3)) {
  console.log(page);
}
// [1, 2, 3]
// [4, 5, 6]
// [7, 8, 9]
// [10]
```

### **3. Tree Traversal**

```javascript
class TreeNode {
  constructor(value, left = null, right = null) {
    this.value = value;
    this.left = left;
    this.right = right;
  }

  *inOrderTraversal() {
    if (this.left) {
      yield* this.left.inOrderTraversal();
    }
    yield this.value;
    if (this.right) {
      yield* this.right.inOrderTraversal();
    }
  }

  *preOrderTraversal() {
    yield this.value;
    if (this.left) {
      yield* this.left.preOrderTraversal();
    }
    if (this.right) {
      yield* this.right.preOrderTraversal();
    }
  }
}

const tree = new TreeNode(
  4,
  new TreeNode(2, new TreeNode(1), new TreeNode(3)),
  new TreeNode(6, new TreeNode(5), new TreeNode(7)),
);

console.log([...tree.inOrderTraversal()]); // [1, 2, 3, 4, 5, 6, 7]
console.log([...tree.preOrderTraversal()]); // [4, 2, 1, 3, 6, 5, 7]
```

### **4. Async Data Streaming**

```javascript
async function* fetchPages(url, maxPages) {
  let page = 1;

  while (page <= maxPages) {
    const response = await fetch(`${url}?page=${page}`);
    const data = await response.json();
    yield data;
    page++;
  }
}

// Usage
(async () => {
  for await (const page of fetchPages("/api/data", 3)) {
    console.log("Received page:", page);
  }
})();
```

---

## 🎯 Custom Iterables

### **Range Object**

```javascript
class Range {
  constructor(start, end, step = 1) {
    this.start = start;
    this.end = end;
    this.step = step;
  }

  *[Symbol.iterator]() {
    for (let i = this.start; i <= this.end; i += this.step) {
      yield i;
    }
  }
}

const range = new Range(1, 10, 2);

console.log([...range]); // [1, 3, 5, 7, 9]

for (const num of range) {
  console.log(num); // 1, 3, 5, 7, 9
}
```

### **Infinite Iterable**

```javascript
class InfiniteCounter {
  constructor(start = 0, step = 1) {
    this.start = start;
    this.step = step;
  }

  *[Symbol.iterator]() {
    let current = this.start;
    while (true) {
      yield current;
      current += this.step;
    }
  }
}

const counter = new InfiniteCounter(0, 5);
const iterator = counter[Symbol.iterator]();

console.log(iterator.next().value); // 0
console.log(iterator.next().value); // 5
console.log(iterator.next().value); // 10
```

---

## 🏗️ Generator Composition

### **Combining Multiple Generators**

```javascript
function* compose(...generators) {
  for (const generator of generators) {
    yield* generator();
  }
}

function* gen1() {
  yield 1;
  yield 2;
}

function* gen2() {
  yield 3;
  yield 4;
}

function* gen3() {
  yield 5;
  yield 6;
}

const combined = compose(gen1, gen2, gen3);
console.log([...combined]); // [1, 2, 3, 4, 5, 6]
```

### **Generator Pipeline**

```javascript
function* map(iterable, fn) {
  for (const item of iterable) {
    yield fn(item);
  }
}

function* filter(iterable, predicate) {
  for (const item of iterable) {
    if (predicate(item)) {
      yield item;
    }
  }
}

function* take(iterable, n) {
  let count = 0;
  for (const item of iterable) {
    if (count++ >= n) break;
    yield item;
  }
}

// Usage
function* numbers() {
  let i = 0;
  while (true) yield i++;
}

const result = take(
  filter(
    map(numbers(), (x) => x * 2),
    (x) => x % 3 === 0,
  ),
  5,
);

console.log([...result]); // [0, 6, 12, 18, 24]
```

---

## 🧪 Interview Questions

### **Q1: What's the difference between an iterator and an iterable?**

**Answer:**

- **Iterable**: An object that implements `Symbol.iterator` and returns an iterator
- **Iterator**: An object with a `next()` method that returns `{value, done}`

### **Q2: Implement a generator that generates unique IDs**

```javascript
function* idGenerator() {
  let id = 1;
  while (true) {
    yield id++;
  }
}

const ids = idGenerator();
console.log(ids.next().value); // 1
console.log(ids.next().value); // 2
```

### **Q3: What happens when you call a generator function?**

**Answer:** Calling a generator function doesn't execute the code immediately. Instead, it returns a generator object that implements the iterator protocol. The code executes only when you call `next()`.

---

## 🎯 Best Practices

1. **Use generators for lazy evaluation** - compute values on-demand
2. **Use `yield*` for delegation** - cleaner than manual iteration
3. **Generators are perfect for sequences** - especially infinite ones
4. **Combine with async/await** for async iteration
5. **Use for...of** to consume iterators naturally
6. **Return early with `return()`** when you're done
7. **Handle errors with try/catch** inside generators

---

## 📚 Key Takeaways

- **Iterators** define sequences using the `next()` method
- **Iterables** implement `Symbol.iterator` and can be used in `for...of`
- **Generators** make creating iterators much easier with `function*` and `yield`
- **`yield*`** delegates to another generator or iterable
- **Generators are lazy** - values are computed on demand
- **Perfect for** sequences, tree traversal, pagination, and streaming data
- **Works with async** - async generators with `for await...of`

---

**Master generators for elegant, memory-efficient code!**
