# Proxies & Reflect API in JavaScript

> **Metaprogramming: Intercept and customize fundamental operations on objects**

---

## 🎯 What is a Proxy?

A **Proxy** wraps an object and intercepts operations performed on it, allowing you to customize behavior for fundamental operations like property access, assignment, function invocation, etc.

```javascript
const target = { name: "Alice", age: 30 };

const handler = {
  get(target, property) {
    console.log(`Getting ${property}`);
    return target[property];
  },
};

const proxy = new Proxy(target, handler);

console.log(proxy.name); // Logs: "Getting name", Returns: "Alice"
```

---

## 📚 Proxy Traps

Proxies can intercept these operations (traps):

| Trap                       | Intercepts                          | Example                                        |
| -------------------------- | ----------------------------------- | ---------------------------------------------- |
| `get`                      | Property access                     | `obj.prop`, `obj['prop']`                      |
| `set`                      | Property assignment                 | `obj.prop = value`                             |
| `has`                      | `in` operator                       | `'prop' in obj`                                |
| `deleteProperty`           | `delete` operator                   | `delete obj.prop`                              |
| `apply`                    | Function call                       | `func()`                                       |
| `construct`                | `new` operator                      | `new Func()`                                   |
| `getPrototypeOf`           | `Object.getPrototypeOf()`           | `Object.getPrototypeOf(obj)`                   |
| `setPrototypeOf`           | `Object.setPrototypeOf()`           | `Object.setPrototypeOf(obj, proto)`            |
| `isExtensible`             | `Object.isExtensible()`             | `Object.isExtensible(obj)`                     |
| `preventExtensions`        | `Object.preventExtensions()`        | `Object.preventExtensions(obj)`                |
| `getOwnPropertyDescriptor` | `Object.getOwnPropertyDescriptor()` | `Object.getOwnPropertyDescriptor(obj, prop)`   |
| `defineProperty`           | `Object.defineProperty()`           | `Object.defineProperty(obj, prop, descriptor)` |
| `ownKeys`                  | `Object.keys()`, `for...in`         | `Object.keys(obj)`                             |

---

## 🔥 Common Proxy Patterns

### **1. Validation**

```javascript
const validator = {
  set(target, property, value) {
    if (property === "age") {
      if (typeof value !== "number" || value < 0) {
        throw new TypeError("Age must be a positive number");
      }
    }
    target[property] = value;
    return true; // Indicates success
  },
};

const person = new Proxy({}, validator);

person.age = 30; // Works
person.age = -5; // Throws TypeError
person.age = "30"; // Throws TypeError
```

### **2. Default Values**

```javascript
const withDefaults = (target, defaults) => {
  return new Proxy(target, {
    get(target, property) {
      return property in target ? target[property] : defaults[property];
    },
  });
};

const settings = withDefaults(
  { theme: "dark" },
  { theme: "light", language: "en", notifications: true },
);

console.log(settings.theme); // "dark"
console.log(settings.language); // "en" (from defaults)
console.log(settings.notifications); // true (from defaults)
```

### **3. Negative Array Indices (Python-style)**

```javascript
const negativeArray = (array) => {
  return new Proxy(array, {
    get(target, property) {
      const index = Number(property);
      if (Number.isInteger(index) && index < 0) {
        return target[target.length + index];
      }
      return target[property];
    },
  });
};

const arr = negativeArray([1, 2, 3, 4, 5]);

console.log(arr[-1]); // 5
console.log(arr[-2]); // 4
console.log(arr[0]); // 1
```

### **4. Read-Only Object**

```javascript
const readOnly = (target) => {
  return new Proxy(target, {
    set() {
      throw new Error("Cannot modify read-only object");
    },
    deleteProperty() {
      throw new Error("Cannot delete from read-only object");
    },
  });
};

const config = readOnly({ API_KEY: "secret123" });

config.API_KEY = "new"; // Throws Error
delete config.API_KEY; // Throws Error
console.log(config.API_KEY); // "secret123"
```

### **5. Property Access Logging**

```javascript
const logger = (target, name = "Object") => {
  return new Proxy(target, {
    get(target, property) {
      console.log(`[${name}] GET ${String(property)}`);
      return target[property];
    },
    set(target, property, value) {
      console.log(`[${name}] SET ${String(property)} = ${value}`);
      target[property] = value;
      return true;
    },
  });
};

const user = logger({ name: "Alice" }, "User");

user.name; // Logs: "[User] GET name"
user.age = 30; // Logs: "[User] SET age = 30"
```

---

## 🎨 Advanced Proxy Patterns

### **1. Observable / Reactive Object**

```javascript
const createObservable = (target, callback) => {
  return new Proxy(target, {
    set(target, property, value) {
      const oldValue = target[property];
      target[property] = value;
      callback(property, oldValue, value);
      return true;
    },
  });
};

const state = createObservable({ count: 0 }, (prop, oldVal, newVal) => {
  console.log(`${prop} changed: ${oldVal} → ${newVal}`);
});

state.count = 1; // Logs: "count changed: 0 → 1"
state.count = 2; // Logs: "count changed: 1 → 2"
```

### **2. Function Call Tracking**

```javascript
const trackCalls = (func, name) => {
  let callCount = 0;

  return new Proxy(func, {
    apply(target, thisArg, args) {
      callCount++;
      console.log(`${name} called ${callCount} time(s)`);
      return target.apply(thisArg, args);
    },
  });
};

const add = trackCalls((a, b) => a + b, "add");

add(1, 2); // Logs: "add called 1 time(s)", Returns: 3
add(3, 4); // Logs: "add called 2 time(s)", Returns: 7
```

### **3. Memoization with Proxy**

```javascript
const memoize = (func) => {
  const cache = new Map();

  return new Proxy(func, {
    apply(target, thisArg, args) {
      const key = JSON.stringify(args);

      if (cache.has(key)) {
        console.log("Returning cached result");
        return cache.get(key);
      }

      const result = target.apply(thisArg, args);
      cache.set(key, result);
      return result;
    },
  });
};

const expensiveFunction = memoize((n) => {
  console.log("Computing...");
  return n * n;
});

expensiveFunction(5); // Logs: "Computing...", Returns: 25
expensiveFunction(5); // Logs: "Returning cached result", Returns: 25
```

### **4. Virtual Properties**

```javascript
const virtualProps = (target) => {
  return new Proxy(target, {
    get(target, property) {
      if (property === "fullName") {
        return `${target.firstName} ${target.lastName}`;
      }
      if (property === "age") {
        const ageDiff = Date.now() - target.birthDate.getTime();
        return Math.floor(ageDiff / (365.25 * 24 * 60 * 60 * 1000));
      }
      return target[property];
    },
  });
};

const person = virtualProps({
  firstName: "John",
  lastName: "Doe",
  birthDate: new Date("1990-01-01"),
});

console.log(person.fullName); // "John Doe"
console.log(person.age); // Calculated age
```

---

## 🪞 Reflect API

The **Reflect** API provides methods for interceptable JavaScript operations. It mirrors Proxy traps and provides a more functional way to perform object operations.

### **Why Use Reflect?**

1. **Returns boolean** for success/failure instead of throwing errors
2. **Functional approach** to object operations
3. **Matches Proxy trap signatures** exactly
4. **More predictable** than operators in some cases

### **Reflect Methods**

```javascript
const obj = { name: "Alice", age: 30 };

// Instead of obj.name
Reflect.get(obj, "name"); // "Alice"

// Instead of obj.name = 'Bob'
Reflect.set(obj, "name", "Bob"); // true

// Instead of 'name' in obj
Reflect.has(obj, "name"); // true

// Instead of delete obj.age
Reflect.deleteProperty(obj, "age"); // true

// Instead of Object.keys(obj)
Reflect.ownKeys(obj); // ['name']

// Instead of new Constructor()
Reflect.construct(Array, [1, 2, 3]); // [1, 2, 3]

// Instead of func.apply(thisArg, args)
Reflect.apply(Math.max, null, [1, 2, 3]); // 3
```

### **Reflect in Proxy Traps**

```javascript
const handler = {
  get(target, property, receiver) {
    console.log(`Getting ${property}`);
    return Reflect.get(target, property, receiver);
  },

  set(target, property, value, receiver) {
    console.log(`Setting ${property} = ${value}`);
    return Reflect.set(target, property, value, receiver);
  },
};

const proxy = new Proxy({ name: "Alice" }, handler);

proxy.name; // Logs: "Getting name"
proxy.age = 30; // Logs: "Setting age = 30"
```

---

## 🏗️ Real-World Use Cases

### **1. API Client with Auto-Retry**

```javascript
const createAPIClient = (baseURL) => {
  return new Proxy(
    {},
    {
      get(target, endpoint) {
        return async function (options = {}) {
          const { retries = 3, ...fetchOptions } = options;

          for (let i = 0; i < retries; i++) {
            try {
              const response = await fetch(
                `${baseURL}/${endpoint}`,
                fetchOptions,
              );
              return await response.json();
            } catch (error) {
              if (i === retries - 1) throw error;
              console.log(`Retry ${i + 1}/${retries}`);
            }
          }
        };
      },
    },
  );
};

const api = createAPIClient("https://api.example.com");

// Usage
const users = await api.users({ method: "GET" });
const posts = await api.posts({ method: "GET", retries: 5 });
```

### **2. Type Checking**

```javascript
const typed = (schema) => {
  return (target) => {
    return new Proxy(target, {
      set(target, property, value) {
        const expectedType = schema[property];

        if (expectedType && typeof value !== expectedType) {
          throw new TypeError(
            `Property ${property} must be of type ${expectedType}`,
          );
        }

        target[property] = value;
        return true;
      },
    });
  };
};

const userSchema = {
  name: "string",
  age: "number",
  active: "boolean",
};

const user = typed(userSchema)({});

user.name = "Alice"; // OK
user.age = 30; // OK
user.age = "30"; // Throws TypeError
```

### **3. Enum Implementation**

```javascript
const createEnum = (...values) => {
  const enumObj = {};

  for (const value of values) {
    enumObj[value] = value;
  }

  return new Proxy(enumObj, {
    set() {
      throw new Error("Cannot modify enum");
    },
    deleteProperty() {
      throw new Error("Cannot delete enum value");
    },
  });
};

const Colors = createEnum("RED", "GREEN", "BLUE");

console.log(Colors.RED); // "RED"
Colors.RED = "YELLOW"; // Throws Error
delete Colors.GREEN; // Throws Error
```

---

## 🧪 Interview Questions

### **Q1: What's the difference between Proxy and Object.defineProperty?**

**Answer:**

- **Proxy**: Intercepts operations on entire object, can trap more operations (13 traps)
- **Object.defineProperty**: Defines behavior for individual properties, limited operations

### **Q2: Can you proxy a proxy?**

**Answer:** Yes! You can create a proxy of a proxy:

```javascript
const target = { value: 42 };
const proxy1 = new Proxy(target, handler1);
const proxy2 = new Proxy(proxy1, handler2);
```

### **Q3: What is Reflect used for?**

**Answer:** Reflect provides:

- Functional approach to object operations
- Return values instead of throwing errors
- Perfect match for Proxy trap signatures
- More predictable behavior

---

## 🎯 Best Practices

1. **Use Reflect in traps** for default behavior
2. **Always return true/false** from `set` and `deleteProperty` traps
3. **Handle errors gracefully** - don't break existing code
4. **Document proxy behavior** - it's not obvious
5. **Be careful with performance** - traps add overhead
6. **Use for frameworks/libraries** more than application code
7. **Consider revocable proxies** for temporary access control

---

## 📚 Key Takeaways

- **Proxies** intercept and customize fundamental object operations
- **13 trap types** cover almost all object operations
- **Reflect API** provides functional approach to object operations
- **Perfect for**: validation, logging, reactive systems, API clients
- **Use cases**: Vue 3 reactivity, mobx, validation libraries
- **Performance consideration**: adds overhead, use judiciously

---

**Master proxies for powerful metaprogramming capabilities!**
