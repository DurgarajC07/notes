# Monads & Functors

> **Functional programming patterns: Maybe, Either, and more**

---

## 🎯 Overview

Functors and Monads are functional programming patterns that provide a structured way to work with values in contexts (like nullable values, errors, async operations). While the terminology can be intimidating, they solve real problems in JavaScript.

---

## 📚 Core Concepts

### **Concept 1: Functors**

A **Functor** is a container that can be mapped over:

```javascript
// Arrays are functors
[1, 2, 3].map((x) => x * 2); // [2, 4, 6]

// A simple functor
class Box {
  constructor(value) {
    this.value = value;
  }

  map(fn) {
    return new Box(fn(this.value));
  }
}

const box = new Box(5);
box
  .map((x) => x * 2) // Box(10)
  .map((x) => x + 1) // Box(11)
  .map((x) => `Value: ${x}`); // Box('Value: 11')
```

**Functor laws:**

1. **Identity:** `F.map(x => x)` equals `F`
2. **Composition:** `F.map(f).map(g)` equals `F.map(x => g(f(x)))`

### **Concept 2: Maybe Monad (Handling Nulls)**

Maybe wraps values that might be null/undefined:

```javascript
class Maybe {
  constructor(value) {
    this.value = value;
  }

  static of(value) {
    return new Maybe(value);
  }

  isNothing() {
    return this.value === null || this.value === undefined;
  }

  map(fn) {
    return this.isNothing() ? this : Maybe.of(fn(this.value));
  }

  flatMap(fn) {
    return this.isNothing() ? this : fn(this.value);
  }

  getOrElse(defaultValue) {
    return this.isNothing() ? defaultValue : this.value;
  }
}

// Without Maybe - verbose null checks
function getStreetName(user) {
  if (user && user.address && user.address.street) {
    return user.address.street.name;
  }
  return "Unknown";
}

// With Maybe - chainable
function getStreetName(user) {
  return Maybe.of(user)
    .map((u) => u.address)
    .map((a) => a.street)
    .map((s) => s.name)
    .getOrElse("Unknown");
}
```

### **Concept 3: Either Monad (Error Handling)**

Either represents a value that's either Right (success) or Left (error):

```javascript
class Either {
  constructor(value) {
    this.value = value;
  }

  static left(value) {
    return new Left(value);
  }

  static right(value) {
    return new Right(value);
  }
}

class Left extends Either {
  map(fn) {
    return this; // Ignore map on Left
  }

  flatMap(fn) {
    return this;
  }

  getOrElse(defaultValue) {
    return defaultValue;
  }

  toString() {
    return `Left(${this.value})`;
  }
}

class Right extends Either {
  map(fn) {
    return Either.right(fn(this.value));
  }

  flatMap(fn) {
    return fn(this.value);
  }

  getOrElse(defaultValue) {
    return this.value;
  }

  toString() {
    return `Right(${this.value})`;
  }
}

// Usage
function parseJSON(str) {
  try {
    return Either.right(JSON.parse(str));
  } catch (error) {
    return Either.left(error.message);
  }
}

parseJSON('{"name": "Alice"}')
  .map((obj) => obj.name)
  .map((name) => name.toUpperCase())
  .getOrElse("Unknown"); // 'ALICE'

parseJSON("invalid json")
  .map((obj) => obj.name)
  .map((name) => name.toUpperCase())
  .getOrElse("Unknown"); // 'Unknown'
```

---

## 🔥 Practical Examples

### **Example 1: Safe Property Access**

```javascript
// Without Maybe
function getUserCity(user) {
  if (user && user.profile && user.profile.address) {
    return user.profile.address.city || "Unknown";
  }
  return "Unknown";
}

// With Maybe
function getUserCity(user) {
  return Maybe.of(user)
    .map((u) => u.profile)
    .map((p) => p.address)
    .map((a) => a.city)
    .getOrElse("Unknown");
}

// Real-world usage
const users = [
  { profile: { address: { city: "NYC" } } },
  { profile: {} },
  null,
];

const cities = users.map(getUserCity);
// ['NYC', 'Unknown', 'Unknown']
```

### **Example 2: Validation Pipeline**

```javascript
function validateEmail(email) {
  if (!email) {
    return Either.left("Email is required");
  }
  if (!email.includes("@")) {
    return Either.left("Email must contain @");
  }
  if (email.length < 5) {
    return Either.left("Email too short");
  }
  return Either.right(email);
}

function validateAge(age) {
  if (age < 18) {
    return Either.left("Must be 18 or older");
  }
  return Either.right(age);
}

// Validation chain
function validateUser(email, age) {
  return validateEmail(email)
    .flatMap(() => validateAge(age))
    .map(() => ({ email, age }));
}

const result1 = validateUser("user@example.com", 25);
// Right({ email: 'user@example.com', age: 25 })

const result2 = validateUser("invalid", 25);
// Left('Email must contain @')

const result3 = validateUser("user@example.com", 15);
// Left('Must be 18 or older')
```

### **Example 3: Task Monad (Async)**

```javascript
class Task {
  constructor(fork) {
    this.fork = fork;
  }

  static of(value) {
    return new Task((reject, resolve) => resolve(value));
  }

  map(fn) {
    return new Task((reject, resolve) =>
      this.fork(reject, (value) => resolve(fn(value))),
    );
  }

  flatMap(fn) {
    return new Task((reject, resolve) =>
      this.fork(reject, (value) => fn(value).fork(reject, resolve)),
    );
  }
}

// Usage
function fetchUser(id) {
  return new Task((reject, resolve) => {
    fetch(`/api/users/${id}`)
      .then((r) => r.json())
      .then(resolve)
      .catch(reject);
  });
}

function fetchPosts(userId) {
  return new Task((reject, resolve) => {
    fetch(`/api/posts?userId=${userId}`)
      .then((r) => r.json())
      .then(resolve)
      .catch(reject);
  });
}

// Compose async operations
fetchUser(1)
  .flatMap((user) => fetchPosts(user.id))
  .map((posts) => posts.filter((p) => p.published))
  .fork(
    (error) => console.error("Failed:", error),
    (posts) => console.log("Posts:", posts),
  );
```

### **Example 4: Real-World API Error Handling**

```javascript
class Result {
  static ok(value) {
    return new Ok(value);
  }

  static err(error) {
    return new Err(error);
  }
}

class Ok {
  constructor(value) {
    this.value = value;
  }

  map(fn) {
    return Result.ok(fn(this.value));
  }

  mapError(fn) {
    return this;
  }

  flatMap(fn) {
    return fn(this.value);
  }
}

class Err {
  constructor(error) {
    this.error = error;
  }

  map(fn) {
    return this;
  }

  mapError(fn) {
    return Result.err(fn(this.error));
  }

  flatMap(fn) {
    return this;
  }
}

// API wrapper
async function apiCall(url) {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      return Result.err({
        status: response.status,
        message: "HTTP Error",
      });
    }
    const data = await response.json();
    return Result.ok(data);
  } catch (error) {
    return Result.err({
      status: 0,
      message: error.message,
    });
  }
}

// Usage
const result = await apiCall("/api/user")
  .then((r) => r.map((user) => user.name))
  .then((r) => r.map((name) => name.toUpperCase()))
  .then((r) => r.mapError((err) => `Failed: ${err.message}`));
```

---

## 🏗️ Best Practices

1. **Use Maybe for nullable values** - Avoid null checks

   ```javascript
   Maybe.of(user)
     .map((u) => u.name)
     .getOrElse("Unknown");
   ```

2. **Use Either for error handling** - Railway-oriented programming

   ```javascript
   parseJSON(str).flatMap(validate).flatMap(save);
   ```

3. **Keep monads simple** - Don't overcomplicate

   ```javascript
   // Sometimes optional chaining is enough
   const city = user?.profile?.address?.city ?? "Unknown";
   ```

4. **Use flatMap for chaining** - Avoid nested monads

   ```javascript
   // ✅ flatMap
   Maybe.of(user).flatMap(getAddress); // Maybe

   // ❌ map creates Maybe<Maybe>
   Maybe.of(user).map(getAddress); // Maybe<Maybe>
   ```

---

## 🧪 Interview Questions

### **Q1: What is a Functor?**

**Answer:** A Functor is a container that implements `map`. It allows you to apply a function to the wrapped value without unwrapping it. Arrays, Promises, and Maybe are all functors.

### **Q2: What problem does the Maybe monad solve?**

**Answer:** Maybe handles null/undefined values elegantly without nested null checks. It chains operations safely – if any step returns null, subsequent operations are skipped.

### **Q3: When would you use Either vs Maybe?**

**Answer:**

- **Maybe:** Simple presence/absence of value
- **Either:** Need to capture error information (Left contains error details)

---

## 📚 Key Takeaways

- Functors are containers that can be mapped over
- Maybe handles nullable values without null checks
- Either handles errors in a functional way (Left = error, Right = success)
- Monads enable safe chaining of operations
- Use flatMap to avoid nested monads
- Modern JavaScript has alternatives (optional chaining, nullish coalescing)
- Monads provide structure for error handling and async operations

---

**Master monads & functors for production-ready code!**
