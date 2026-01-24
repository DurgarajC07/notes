# Symbols & WeakMaps in JavaScript

> **Private properties, unique identifiers, and memory-efficient object storage**

---

## 🎯 What are Symbols?

**Symbols** are primitive values that are guaranteed to be unique. They're often used to create private object properties or unique identifiers.

```javascript
const sym1 = Symbol();
const sym2 = Symbol();

console.log(sym1 === sym2); // false - always unique

const sym3 = Symbol("description");
console.log(sym3.toString()); // "Symbol(description)"
```

---

## 📚 Symbol Features

### **1. Unique Property Keys**

```javascript
const ID = Symbol("id");
const user = {
  name: "Alice",
  [ID]: 12345,
};

console.log(user.name); // "Alice"
console.log(user[ID]); // 12345
console.log(user.ID); // undefined

// Symbol properties are hidden from normal iteration
console.log(Object.keys(user)); // ['name']
console.log(JSON.stringify(user)); // '{"name":"Alice"}'
console.log(Object.getOwnPropertySymbols(user)); // [Symbol(id)]
```

### **2. Global Symbol Registry**

```javascript
// Create or retrieve global symbol
const sym1 = Symbol.for("app.id");
const sym2 = Symbol.for("app.id");

console.log(sym1 === sym2); // true - same symbol from registry

// Get key from symbol
console.log(Symbol.keyFor(sym1)); // "app.id"

// Local symbols don't have keys
const localSym = Symbol("local");
console.log(Symbol.keyFor(localSym)); // undefined
```

### **3. Well-Known Symbols**

JavaScript has built-in symbols that customize object behavior:

```javascript
// Symbol.iterator
const iterable = {
  [Symbol.iterator]() {
    let i = 0;
    return {
      next() {
        return i < 3 ? { value: i++, done: false } : { done: true };
      },
    };
  },
};

console.log([...iterable]); // [0, 1, 2]

// Symbol.toStringTag
class MyClass {
  get [Symbol.toStringTag]() {
    return "MyClass";
  }
}

const instance = new MyClass();
console.log(Object.prototype.toString.call(instance)); // "[object MyClass]"

// Symbol.toPrimitive
const obj = {
  [Symbol.toPrimitive](hint) {
    if (hint === "number") return 42;
    if (hint === "string") return "forty-two";
    return true;
  },
};

console.log(+obj); // 42
console.log(`${obj}`); // "forty-two"
console.log(obj + ""); // "true"
```

---

## 🔒 Private Properties with Symbols

```javascript
const _private = Symbol("private");

class BankAccount {
  constructor(balance) {
    this[_private] = { balance };
  }

  deposit(amount) {
    this[_private].balance += amount;
  }

  getBalance() {
    return this[_private].balance;
  }
}

const account = new BankAccount(1000);
account.deposit(500);

console.log(account.getBalance()); // 1500
console.log(account[_private]); // undefined (symbol not in scope)
console.log(Object.keys(account)); // []
console.log(Object.getOwnPropertySymbols(account)); // [Symbol(private)]
```

**Note:** Symbols provide privacy from accidental access, but not true privacy (can still be accessed via `Object.getOwnPropertySymbols`). For true private fields, use `#` syntax:

```javascript
class BankAccount {
  #balance;

  constructor(balance) {
    this.#balance = balance;
  }

  deposit(amount) {
    this.#balance += amount;
  }

  getBalance() {
    return this.#balance;
  }
}
```

---

## 🗺️ WeakMap

A **WeakMap** is a collection of key-value pairs where keys must be objects and are held weakly (can be garbage collected).

### **Key Characteristics**

- Keys must be objects
- Keys are held weakly (garbage collection)
- Not enumerable
- No `size` property
- No `clear()` method

```javascript
const weakMap = new WeakMap();

let obj = { name: "Alice" };

weakMap.set(obj, "some data");
console.log(weakMap.get(obj)); // "some data"
console.log(weakMap.has(obj)); // true

// When obj is no longer referenced, it can be garbage collected
obj = null; // Data in WeakMap will be garbage collected
```

---

## 🎯 WeakMap Use Cases

### **1. Private Data Storage**

```javascript
const privateData = new WeakMap();

class User {
  constructor(name, password) {
    privateData.set(this, { password });
    this.name = name;
  }

  authenticate(password) {
    return privateData.get(this).password === password;
  }

  changePassword(oldPassword, newPassword) {
    if (this.authenticate(oldPassword)) {
      privateData.get(this).password = newPassword;
      return true;
    }
    return false;
  }
}

const user = new User("Alice", "secret123");

console.log(user.name); // "Alice"
console.log(user.password); // undefined
console.log(user.authenticate("secret123")); // true
console.log(user.changePassword("secret123", "newSecret")); // true
```

### **2. Caching/Memoization**

```javascript
const cache = new WeakMap();

function expensiveOperation(obj) {
  if (cache.has(obj)) {
    console.log("Returning cached result");
    return cache.get(obj);
  }

  console.log("Computing...");
  const result = obj.value * 2; // Expensive calculation
  cache.set(obj, result);
  return result;
}

const obj1 = { value: 5 };
console.log(expensiveOperation(obj1)); // Computing... 10
console.log(expensiveOperation(obj1)); // Returning cached result 10
```

### **3. Metadata Storage**

```javascript
const metadata = new WeakMap();

class Element {
  constructor(type) {
    this.type = type;
    metadata.set(this, {
      created: new Date(),
      accessCount: 0,
    });
  }

  access() {
    const meta = metadata.get(this);
    meta.accessCount++;
    meta.lastAccessed = new Date();
  }

  getMetadata() {
    return { ...metadata.get(this) };
  }
}

const element = new Element("div");
element.access();
element.access();

console.log(element.getMetadata());
// { created: ..., accessCount: 2, lastAccessed: ... }
```

### **4. DOM Node Data**

```javascript
const nodeData = new WeakMap();

function attachData(node, data) {
  nodeData.set(node, data);
}

function getData(node) {
  return nodeData.get(node);
}

const button = document.createElement("button");
attachData(button, { clicks: 0, handler: () => console.log("clicked") });

button.addEventListener("click", () => {
  const data = getData(button);
  data.clicks++;
});

// When button is removed from DOM and no longer referenced,
// associated data will be automatically garbage collected
```

---

## 🗑️ WeakSet

Similar to WeakMap, but stores only objects (not key-value pairs):

```javascript
const visitedNodes = new WeakSet();

function traverse(node) {
  if (visitedNodes.has(node)) {
    return; // Already visited
  }

  visitedNodes.add(node);

  // Process node
  console.log(node.value);

  // Traverse children
  if (node.left) traverse(node.left);
  if (node.right) traverse(node.right);
}

const tree = {
  value: 1,
  left: { value: 2 },
  right: { value: 3 },
};

traverse(tree);
```

---

## 🆚 Map vs WeakMap

| Feature            | Map             | WeakMap               |
| ------------------ | --------------- | --------------------- |
| Key types          | Any             | Objects only          |
| Garbage collection | No              | Yes                   |
| Enumerable         | Yes             | No                    |
| `.size`            | Yes             | No                    |
| `.clear()`         | Yes             | No                    |
| Iteration          | Yes             | No                    |
| Use case           | General storage | Private data, caching |

```javascript
// Map
const map = new Map();
map.set("key", "value");
map.set(123, "number key");
console.log(map.size); // 2

// WeakMap
const weakMap = new WeakMap();
// weakMap.set('key', 'value'); // Error: Invalid value used as weak map key
weakMap.set({}, "value"); // OK
// console.log(weakMap.size); // undefined
```

---

## 🔧 Combining Symbols and WeakMaps

```javascript
const PRIVATE = Symbol("private");

class SecureClass {
  constructor(secret) {
    // Use WeakMap for truly private data
    const privateData = new WeakMap();
    privateData.set(this, { secret });

    // Store reference using Symbol
    this[PRIVATE] = privateData;
  }

  getSecret() {
    return this[PRIVATE].get(this).secret;
  }
}
```

---

## 🏗️ Real-World Patterns

### **1. Event Emitter with WeakMap**

```javascript
const listeners = new WeakMap();

class EventEmitter {
  on(event, callback) {
    if (!listeners.has(this)) {
      listeners.set(this, {});
    }

    const events = listeners.get(this);
    if (!events[event]) {
      events[event] = [];
    }

    events[event].push(callback);
  }

  emit(event, ...args) {
    const events = listeners.get(this);
    if (events && events[event]) {
      events[event].forEach((callback) => callback(...args));
    }
  }
}

const emitter = new EventEmitter();
emitter.on("data", (data) => console.log("Received:", data));
emitter.emit("data", "Hello"); // Logs: "Received: Hello"
```

### **2. Reactive System**

```javascript
const reactiveData = new WeakMap();

function reactive(obj) {
  const handlers = new Set();

  reactiveData.set(obj, handlers);

  return new Proxy(obj, {
    set(target, property, value) {
      target[property] = value;
      handlers.forEach((handler) => handler(property, value));
      return true;
    },
  });
}

function watch(obj, handler) {
  const handlers = reactiveData.get(obj);
  if (handlers) {
    handlers.add(handler);
  }
}

const state = reactive({ count: 0 });

watch(state, (prop, value) => {
  console.log(`${prop} changed to ${value}`);
});

state.count = 1; // Logs: "count changed to 1"
state.count = 2; // Logs: "count changed to 2"
```

---

## 🧪 Interview Questions

### **Q1: Why use WeakMap instead of Map?**

**Answer:** WeakMap allows garbage collection of keys. Use it when:

- You want to store data associated with objects without preventing GC
- You need private data storage
- You're caching data for objects

### **Q2: Can you iterate over a WeakMap?**

**Answer:** No. WeakMaps are not enumerable because keys can be garbage collected at any time, making iteration unreliable.

### **Q3: What's the difference between Symbol() and Symbol.for()?**

**Answer:**

- `Symbol()` creates a new unique symbol every time
- `Symbol.for(key)` creates or retrieves a global symbol from the registry

---

## 🎯 Best Practices

1. **Use Symbols for unique property keys** that won't conflict
2. **Use WeakMap for private data** associated with objects
3. **Use Symbol.for() sparingly** - only when you need global symbols
4. **Document Symbol usage** - they're not obvious
5. **Prefer # private fields** over Symbols for true privacy
6. **Use WeakMap for caching** to avoid memory leaks
7. **Remember WeakMap keys must be objects** - primitives not allowed

---

## 📚 Key Takeaways

- **Symbols** create unique, non-conflicting property keys
- **Well-known Symbols** customize object behavior
- **WeakMap** stores object-keyed data with garbage collection
- **WeakSet** stores objects without preventing GC
- **Use cases**: private data, caching, metadata, DOM associations
- **Symbols ≠ true privacy** - use `#` private fields for that
- **WeakMap prevents memory leaks** in caching scenarios

---

**Leverage Symbols and WeakMaps for elegant, memory-efficient code!**
