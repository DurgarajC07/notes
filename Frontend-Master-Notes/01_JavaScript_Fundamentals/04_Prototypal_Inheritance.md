# 🔗 Prototypal Inheritance

> JavaScript uses prototypal inheritance, not classical inheritance. Understanding the prototype chain is essential for mastering object-oriented JavaScript and avoiding common pitfalls.

---

## 📖 1. Concept Explanation

### What is Prototypal Inheritance?

**Prototypal inheritance** means objects inherit directly from other objects through a prototype chain. Every object has an internal `[[Prototype]]` link to another object (or `null`).

```javascript
const animal = {
  eat() {
    console.log("Eating...");
  },
};

const dog = Object.create(animal);
dog.bark = function () {
  console.log("Woof!");
};

dog.eat(); // 'Eating...' (inherited from animal)
dog.bark(); // 'Woof!' (own property)
```

### Prototype Chain

When accessing a property:

1. Check if object has the property (own property)
2. If not, check object's prototype
3. If not, check prototype's prototype
4. Continue until reaching `null`

```javascript
dog.eat
  ↓ not found on dog
  ↓ check dog.[[Prototype]] → animal
  ↓ found! animal.eat
```

### Key Concepts

**1. `__proto__` vs `prototype`**

```javascript
function User(name) {
  this.name = name;
}

User.prototype.greet = function () {
  return `Hello, ${this.name}`;
};

const user = new User("Alice");

// Different things:
user.__proto__ === User.prototype; // true - instance's prototype
User.prototype.constructor === User; // true - points back to constructor
User.__proto__ === Function.prototype; // true - functions are objects too
```

**2. Constructor Functions**

```javascript
function Animal(name) {
  this.name = name; // Instance property
}

// Prototype method (shared)
Animal.prototype.speak = function () {
  console.log(`${this.name} makes a sound`);
};

const cat = new Animal("Cat");
cat.speak(); // 'Cat makes a sound'
```

**3. ES6 Classes (Syntactic Sugar)**

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    console.log(`${this.name} makes a sound`);
  }
}

// Under the hood, same as constructor function:
// Animal.prototype.speak = function() { ... }
```

---

## 🧠 2. Why It Matters in Real Projects

### Memory Efficiency

```javascript
// ❌ BAD: Each instance gets own copy of method
function User(name) {
  this.name = name;
  this.greet = function () {
    // New function for EACH instance
    return `Hello, ${this.name}`;
  };
}

const users = Array(1000)
  .fill(0)
  .map((_, i) => new User(`User${i}`));
// 1000 separate greet functions in memory!

// ✅ GOOD: All instances share prototype method
function User(name) {
  this.name = name;
}

User.prototype.greet = function () {
  // ONE function shared by all
  return `Hello, ${this.name}`;
};

const users = Array(1000)
  .fill(0)
  .map((_, i) => new User(`User${i}`));
// Only 1 greet function in memory!
```

### Framework Patterns

```javascript
// React: Component inheritance (deprecated, but shows prototype usage)
class MyComponent extends React.Component {
  render() {
    return <div>Hello</div>;
  }
}

// MyComponent.prototype.__proto__ === React.Component.prototype
// Inherits lifecycle methods, setState, etc.
```

### Production Impact

**1. Performance:**

- Prototype methods faster than closures (less memory)
- Shared methods = better memory usage

**2. Extensibility:**

- Can add methods to built-in prototypes (carefully!)
- Polyfills work through prototypes

**3. Debugging:**

- Understanding prototype chain helps debug "is not a function" errors
- Know when methods come from prototypes vs own properties

---

## ⚙️ 3. Internal Working

### How `new` Works

```javascript
function User(name) {
  this.name = name;
}

const user = new User("Alice");

// What happens internally:
function createInstance(Constructor, ...args) {
  // 1. Create empty object with Constructor.prototype as [[Prototype]]
  const instance = Object.create(Constructor.prototype);

  // 2. Call constructor with 'this' bound to new object
  const result = Constructor.apply(instance, args);

  // 3. Return object (or constructor's return value if it's an object)
  return result && typeof result === "object" ? result : instance;
}
```

### Prototype Chain Resolution

```javascript
const obj = {
  a: 1
};

obj.toString(); // Where does toString come from?

// Chain traversal:
obj.toString
  ↓ not found on obj
  ↓ check obj.__proto__ (Object.prototype)
  ↓ found! Object.prototype.toString
  ↓ call it with this = obj

// Full chain:
obj → Object.prototype → null
```

### Property Lookup Performance

```javascript
// Own property: O(1)
obj.ownProp;

// Prototype property: O(n) where n = prototype chain depth
obj.prototypeMethod; // Checks obj, then obj.__proto__, etc.

// V8 optimization: Inline caches
// After first lookup, V8 remembers where property was found
// Subsequent lookups are fast
```

### Hidden Classes and Shapes

```javascript
// V8 creates "hidden classes" (shapes) for objects
function Point(x, y) {
  this.x = x; // Shape: { x }
  this.y = y; // Shape: { x, y }
}

// Same shape = optimized
const p1 = new Point(1, 2); // Shape: { x, y }
const p2 = new Point(3, 4); // Shape: { x, y } (same!)

// Different shape = deoptimized
const p3 = new Point(5, 6);
p3.z = 7; // Shape: { x, y, z } (different!)
```

---

## ✅ 4. Best Practices

### DO ✅

```javascript
// 1. Use Object.create for clean inheritance
const animal = {
  eat() {
    console.log("Eating");
  },
};

const dog = Object.create(animal);
dog.bark = function () {
  console.log("Woof");
};

// 2. Use ES6 classes for clarity
class Animal {
  constructor(name) {
    this.name = name;
  }

  eat() {
    console.log(`${this.name} is eating`);
  }
}

class Dog extends Animal {
  bark() {
    console.log("Woof!");
  }
}

// 3. Check own properties with hasOwnProperty
const obj = Object.create({ inherited: 1 });
obj.own = 2;

obj.hasOwnProperty("own"); // true
obj.hasOwnProperty("inherited"); // false

// 4. Use Object.getPrototypeOf (not __proto__)
const proto = Object.getPrototypeOf(obj); // ✅ Standard
const proto2 = obj.__proto__; // ❌ Deprecated (but widely used)

// 5. Set prototype only once (performance)
function User(name) {
  this.name = name;
}

// ✅ Set prototype methods ONCE
User.prototype.greet = function () {};
User.prototype.goodbye = function () {};

// ❌ Don't reassign prototype after instances created
const user1 = new User("Alice");
User.prototype = { newMethod() {} }; // user1 still has old prototype!
```

### DON'T ❌

```javascript
// 1. Don't modify built-in prototypes (usually)
// ❌ DANGEROUS
Array.prototype.myMethod = function () {};
// Pollutes global namespace, breaks other code

// ✅ Exception: Polyfills
if (!Array.prototype.includes) {
  Array.prototype.includes = function (searchElement) {
    // Polyfill implementation
  };
}

// 2. Don't use __proto__ in production
// ❌ BAD
obj.__proto__ = prototype;

// ✅ GOOD
Object.setPrototypeOf(obj, prototype);
// Or better: Object.create(prototype)

// 3. Don't create deep prototype chains
// ❌ SLOW
const a = {};
const b = Object.create(a);
const c = Object.create(b);
const d = Object.create(c);
const e = Object.create(d); // 5 levels deep!

// ✅ FLAT
const obj = {
  ...methodsA,
  ...methodsB,
  ...methodsC,
};

// 4. Don't mix constructor and class syntax
// ❌ CONFUSING
function Animal(name) {
  this.name = name;
}

class Dog extends Animal {} // Don't mix!

// ✅ CONSISTENT
class Animal {
  constructor(name) {
    this.name = name;
  }
}

class Dog extends Animal {}
```

---

## ❌ 5. Common Mistakes

### Mistake #1: Forgetting `new`

```javascript
function User(name) {
  this.name = name;
}

// ❌ FORGOT 'new'
const user = User("Alice");
console.log(user); // undefined
console.log(window.name); // 'Alice' (polluted global!)

// ✅ FIX 1: Use 'new'
const user = new User("Alice");

// ✅ FIX 2: Class (throws error without 'new')
class User {
  constructor(name) {
    this.name = name;
  }
}

User("Alice"); // TypeError: Class constructor User cannot be invoked without 'new'

// ✅ FIX 3: Factory function
function createUser(name) {
  return {
    name,
    greet() {
      return `Hello, ${this.name}`;
    },
  };
}
```

### Mistake #2: Shadowing Prototype Methods

```javascript
function User(name) {
  this.name = name;
}

User.prototype.greet = function () {
  return `Hello, ${this.name}`;
};

const user = new User("Alice");

// ❌ Accidentally shadow prototype method
user.greet = "Not a function!"; // Own property shadows prototype

user.greet(); // TypeError: user.greet is not a function

// ✅ Always check before overriding
if (typeof user.greet === "function") {
  // Safe to call
}
```

### Mistake #3: Modifying Prototype After Instantiation

```javascript
function User(name) {
  this.name = name;
}

const user1 = new User("Alice");

// Add method to prototype
User.prototype.greet = function () {
  return `Hello, ${this.name}`;
};

user1.greet(); // Works! (instances get new prototype methods)

// ❌ But reassigning prototype doesn't affect existing instances
User.prototype = {
  goodbye() {
    return `Goodbye, ${this.name}`;
  },
};

user1.goodbye(); // TypeError: user1.goodbye is not a function
// user1.__proto__ still points to OLD prototype

const user2 = new User("Bob");
user2.goodbye(); // Works (uses NEW prototype)
```

### Mistake #4: Not Calling Super Constructor

```javascript
// ❌ WRONG
class Animal {
  constructor(name) {
    this.name = name;
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    // ❌ FORGOT super()!
    this.breed = breed; // ReferenceError: Must call super constructor
  }
}

// ✅ CORRECT
class Dog extends Animal {
  constructor(name, breed) {
    super(name); // Call parent constructor
    this.breed = breed; // Now 'this' is available
  }
}
```

---

## 🔐 6. Security Considerations

### Prototype Pollution

```javascript
// ❌ VULNERABLE: User input modifies prototype
function merge(target, source) {
  for (let key in source) {
    target[key] = source[key]; // DANGEROUS!
  }
  return target;
}

const userInput = JSON.parse('{"__proto__": {"isAdmin": true}}');
merge({}, userInput);

// Now ALL objects have isAdmin!
const obj = {};
console.log(obj.isAdmin); // true (pollution!)

// ✅ SAFE: Check for __proto__
function safeMerge(target, source) {
  for (let key in source) {
    if (key === "__proto__" || key === "constructor" || key === "prototype") {
      continue; // Skip dangerous keys
    }
    target[key] = source[key];
  }
  return target;
}

// ✅ SAFER: Use Object.create(null)
const obj = Object.create(null); // No prototype chain
obj.__proto__ = { isAdmin: true };
console.log(obj.isAdmin); // undefined (safe!)
```

### Constructor Injection

```javascript
// ❌ VULNERABLE
const UserSchema = {
  constructor: User, // Attacker can inject
  name: String,
};

// ✅ SAFE: Don't allow constructor in user input
function validateSchema(schema) {
  const dangerous = ["constructor", "__proto__", "prototype"];
  for (let key in schema) {
    if (dangerous.includes(key)) {
      throw new Error(`Dangerous property: ${key}`);
    }
  }
}
```

---

## 🚀 7. Performance Optimization

### 1. Avoid Changing Prototype After Creation

```javascript
// ❌ SLOW: Changes prototype shape
function User(name) {
  this.name = name;
}

const users = [];
for (let i = 0; i < 10000; i++) {
  const user = new User(`User${i}`);
  users.push(user);

  // ❌ Changes each instance's hidden class
  user.id = i;
}

// ✅ FAST: Initialize all properties in constructor
function User(name, id) {
  this.name = name;
  this.id = id; // Same shape for all instances
}

const users = [];
for (let i = 0; i < 10000; i++) {
  users.push(new User(`User${i}`, i));
}
```

### 2. Cache Prototype Methods

```javascript
// ❌ SLOWER: Prototype lookup on every call
for (let i = 0; i < 10000; i++) {
  users[i].greet(); // Prototype lookup each time
}

// ✅ FASTER: Cache method reference
const greet = User.prototype.greet;
for (let i = 0; i < 10000; i++) {
  greet.call(users[i]); // Direct call, no lookup
}
```

### 3. Use Object.create(null) for Dictionaries

```javascript
// ❌ SLOWER: Has prototype chain
const dict = {};
dict["hasOwnProperty"] = "value"; // Shadows method
console.log(dict.hasOwnProperty("key")); // Broken!

// ✅ FASTER: No prototype overhead
const dict = Object.create(null);
dict["hasOwnProperty"] = "value"; // Fine, no conflict
console.log("key" in dict); // Use 'in' operator instead
```

---

## 🧪 8. Code Examples

### Example 1: Classical Inheritance Pattern

```javascript
// Parent class
function Animal(name) {
  this.name = name;
}

Animal.prototype.eat = function () {
  console.log(`${this.name} is eating`);
};

// Child class
function Dog(name, breed) {
  Animal.call(this, name); // Call parent constructor
  this.breed = breed;
}

// Set up inheritance
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

Dog.prototype.bark = function () {
  console.log("Woof!");
};

// Usage
const dog = new Dog("Buddy", "Golden Retriever");
dog.eat(); // 'Buddy is eating' (inherited)
dog.bark(); // 'Woof!' (own method)
```

### Example 2: ES6 Classes

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }

  eat() {
    console.log(`${this.name} is eating`);
  }

  static create(name) {
    return new Animal(name);
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);
    this.breed = breed;
  }

  bark() {
    console.log("Woof!");
  }

  eat() {
    super.eat(); // Call parent method
    console.log("(dog is eating)");
  }
}

const dog = new Dog("Buddy", "Golden Retriever");
dog.eat();
// 'Buddy is eating'
// '(dog is eating)'
```

### Example 3: Composition Over Inheritance

```javascript
// ❌ Deep inheritance hierarchy
class Animal {}
class Mammal extends Animal {}
class Dog extends Mammal {}
class GoldenRetriever extends Dog {}

// ✅ Composition (more flexible)
const canEat = (state) => ({
  eat() {
    console.log(`${state.name} is eating`);
  },
});

const canWalk = (state) => ({
  walk() {
    console.log(`${state.name} is walking`);
  },
});

const canBark = (state) => ({
  bark() {
    console.log("Woof!");
  },
});

const createDog = (name, breed) => {
  const state = {
    name,
    breed,
  };

  return Object.assign({}, canEat(state), canWalk(state), canBark(state));
};

const dog = createDog("Buddy", "Golden Retriever");
dog.eat();
dog.bark();
```

### Example 4: Mixin Pattern

```javascript
// Mixin functions
const TimestampMixin = {
  setTimestamp() {
    this.createdAt = new Date();
  },
  getAge() {
    return Date.now() - this.createdAt;
  },
};

const SerializeMixin = {
  toJSON() {
    return JSON.stringify(this);
  },
  fromJSON(json) {
    Object.assign(this, JSON.parse(json));
  },
};

// Apply mixins
class User {
  constructor(name) {
    this.name = name;
  }
}

Object.assign(User.prototype, TimestampMixin, SerializeMixin);

const user = new User("Alice");
user.setTimestamp();
console.log(user.toJSON());
```

---

## 🏗️ 9. Real-World Use Cases

### Use Case 1: React Component Hierarchy (Legacy)

```javascript
// Old React class components use prototypal inheritance
class Component {
  constructor(props) {
    this.props = props;
    this.state = {};
  }

  setState(newState) {
    this.state = { ...this.state, ...newState };
    this.forceUpdate();
  }

  render() {
    throw new Error("render() must be implemented");
  }
}

class MyComponent extends Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
  }

  render() {
    return `Count: ${this.state.count}`;
  }
}

// Prototype chain:
// myComponent → MyComponent.prototype → Component.prototype → Object.prototype
```

### Use Case 2: Custom Error Classes

```javascript
class APIError extends Error {
  constructor(message, statusCode, endpoint) {
    super(message);
    this.name = "APIError";
    this.statusCode = statusCode;
    this.endpoint = endpoint;

    // Capture stack trace
    if (Error.captureStackTrace) {
      Error.captureStackTrace(this, APIError);
    }
  }

  toJSON() {
    return {
      name: this.name,
      message: this.message,
      statusCode: this.statusCode,
      endpoint: this.endpoint,
    };
  }
}

class NotFoundError extends APIError {
  constructor(endpoint) {
    super("Resource not found", 404, endpoint);
    this.name = "NotFoundError";
  }
}

// Usage
try {
  throw new NotFoundError("/api/users/123");
} catch (error) {
  if (error instanceof APIError) {
    console.log(error.toJSON());
  }
}
```

### Use Case 3: Plugin System

```javascript
class Editor {
  constructor() {
    this.content = "";
    this.plugins = [];
  }

  use(plugin) {
    this.plugins.push(plugin);
    // Extend editor with plugin methods
    Object.assign(this, plugin);
    if (plugin.init) {
      plugin.init.call(this);
    }
  }

  setContent(content) {
    this.content = content;
  }
}

// Plugins
const MarkdownPlugin = {
  name: "markdown",

  init() {
    console.log("Markdown plugin initialized");
  },

  toMarkdown() {
    return this.content; // Access editor's content
  },
};

const SpellCheckPlugin = {
  name: "spellcheck",

  check() {
    console.log("Checking spelling...");
    // Check this.content
  },
};

// Usage
const editor = new Editor();
editor.use(MarkdownPlugin);
editor.use(SpellCheckPlugin);

editor.setContent("# Hello World");
console.log(editor.toMarkdown());
editor.check();
```

---

## ❓ 10. Interview Questions

### Q1: Explain prototypal inheritance in JavaScript.

**Answer:**

Prototypal inheritance means objects inherit properties and methods directly from other objects through a prototype chain.

**Key points:**

1. **Every object has a `[[Prototype]]`** (internal link to another object)
2. **Property lookup traverses the chain** until found or reaching `null`
3. **Different from classical inheritance** (classes don't exist in JavaScript at the engine level)

```javascript
const animal = {
  eat() {
    console.log("Eating");
  },
};

const dog = Object.create(animal);
dog.bark = function () {
  console.log("Woof");
};

dog.eat(); // Looks up chain: dog → animal → found!
```

**Prototype chain:**

```
dog → animal → Object.prototype → null
```

**Follow-up:** How do you check the prototype chain?

```javascript
Object.getPrototypeOf(dog) === animal; // true
dog.isPrototypeOf(animal); // false
animal.isPrototypeOf(dog); // true
dog instanceof Object; // true
```

---

### Q2: What's the difference between `__proto__` and `prototype`?

**Answer:**

| Aspect        | `__proto__`            | `prototype`              |
| ------------- | ---------------------- | ------------------------ |
| **On**        | Every object           | Functions only           |
| **Points to** | Object's prototype     | Object for new instances |
| **Use**       | Access prototype chain | Set up inheritance       |
| **Standard**  | Deprecated (but works) | Standard                 |

```javascript
function User(name) {
  this.name = name;
}

User.prototype.greet = function () {
  return `Hello, ${this.name}`;
};

const user = new User("Alice");

// Different things:
user.__proto__ === User.prototype; // true
User.prototype.constructor === User; // true
User.__proto__ === Function.prototype; // true
Object.getPrototypeOf(user) === User.prototype; // true (standard way)
```

**Memory structure:**

```
user (instance)
  ├─ name: 'Alice' (own property)
  └─ __proto__ → User.prototype
                  ├─ greet: [Function]
                  ├─ constructor: User
                  └─ __proto__ → Object.prototype
```

---

### Q3: What happens when you call a function with `new`?

**Answer:**

Four things happen:

1. **Create new object** with constructor's prototype
2. **Bind `this`** to the new object
3. **Execute constructor** function
4. **Return** the object (unless constructor returns an object)

```javascript
function User(name) {
  this.name = name;
  // Implicit: return this;
}

const user = new User("Alice");

// Equivalent to:
const user = Object.create(User.prototype);
User.call(user, "Alice");
```

**Edge case:**

```javascript
function User(name) {
  this.name = name;
  return { custom: "object" }; // Explicit return
}

const user = new User("Alice");
console.log(user.name); // undefined
console.log(user.custom); // 'object'
// Constructor's return value used instead
```

**Follow-up:** What if you forget `new`?

```javascript
function User(name) {
  this.name = name; // 'this' = window/global!
}

const user = User("Alice"); // ❌ Forgot 'new'
console.log(user); // undefined
console.log(window.name); // 'Alice' (global pollution!)

// Classes throw error:
class User {
  constructor(name) {
    this.name = name;
  }
}

User("Alice"); // TypeError: Cannot call a class as a function
```

---

### Q4: How do you implement inheritance without ES6 classes?

**Answer:**

Three steps:

```javascript
// 1. Parent constructor
function Animal(name) {
  this.name = name;
}

Animal.prototype.eat = function () {
  console.log(`${this.name} is eating`);
};

// 2. Child constructor
function Dog(name, breed) {
  Animal.call(this, name); // Call parent constructor
  this.breed = breed;
}

// 3. Set up prototype chain
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog; // Fix constructor reference

Dog.prototype.bark = function () {
  console.log("Woof!");
};

// Test
const dog = new Dog("Buddy", "Golden Retriever");
dog.eat(); // 'Buddy is eating' (inherited)
dog.bark(); // 'Woof!' (own method)

console.log(dog instanceof Dog); // true
console.log(dog instanceof Animal); // true
```

**Why each step matters:**

1. **`Animal.call(this, name)`** - Initialize parent properties on child instance
2. **`Object.create(Animal.prototype)`** - Set up prototype chain (not `new Animal()`)
3. **`Dog.prototype.constructor = Dog`** - Fix constructor reference (optional but good practice)

---

### Q5: What is prototype pollution and how do you prevent it?

**Answer:**

**Prototype pollution** is when an attacker modifies `Object.prototype` or other prototypes, affecting all objects.

**Attack example:**

```javascript
// Vulnerable merge function
function merge(target, source) {
  for (let key in source) {
    target[key] = source[key]; // DANGEROUS!
  }
}

// Attacker payload
const malicious = JSON.parse('{"__proto__": {"isAdmin": true}}');
merge({}, malicious);

// Now ALL objects are affected
const user = {};
console.log(user.isAdmin); // true (polluted!)
```

**Prevention strategies:**

1. **Validate keys:**

```javascript
function safeMerge(target, source) {
  const dangerous = ["__proto__", "constructor", "prototype"];

  for (let key in source) {
    if (dangerous.includes(key)) continue;
    if (source.hasOwnProperty(key)) {
      target[key] = source[key];
    }
  }
}
```

2. **Use Object.create(null):**

```javascript
const dict = Object.create(null); // No prototype
dict.__proto__ = { isAdmin: true };
console.log(dict.isAdmin); // undefined (safe!)
```

3. **Use Map instead of objects:**

```javascript
const map = new Map();
map.set("__proto__", "safe"); // No pollution
```

4. **Object.freeze:**

```javascript
Object.freeze(Object.prototype); // Prevent modifications
```

---

## 🧩 11. Design Patterns

### Pattern 1: Constructor Pattern

```javascript
function User(name, email) {
  this.name = name;
  this.email = email;
}

User.prototype.getName = function () {
  return this.name;
};

User.prototype.setName = function (name) {
  this.name = name;
};
```

**Pros:** Simple, memory efficient  
**Cons:** Verbose, `new` required

---

### Pattern 2: Factory Pattern

```javascript
function createUser(name, email) {
  return {
    name,
    email,
    getName() {
      return this.name;
    },
    setName(newName) {
      this.name = newName;
    },
  };
}

const user = createUser("Alice", "alice@example.com");
```

**Pros:** No `new` required, flexible  
**Cons:** Methods not shared (memory overhead)

---

### Pattern 3: OLOO (Objects Linked to Other Objects)

```javascript
const UserProto = {
  init(name, email) {
    this.name = name;
    this.email = email;
    return this;
  },

  getName() {
    return this.name;
  },
};

const user = Object.create(UserProto).init("Alice", "alice@example.com");
```

**Pros:** Simple, no `new`, no constructors  
**Cons:** Less familiar to developers from classical languages

---

## 🎯 Summary

Prototypal inheritance is JavaScript's object model:

- **Objects inherit from objects** (not classes)
- **Prototype chain** enables property lookup
- **Memory efficient** (shared methods on prototype)
- **Flexible** (can modify at runtime)

**Key takeaways:**

1. Use `Object.create()` for clean inheritance
2. Prefer ES6 classes for readability
3. Understand `new` keyword behavior
4. Beware prototype pollution
5. Consider composition over inheritance

**Production tips:**

- Initialize all properties in constructor (same shape)
- Don't modify prototypes after instances created
- Use `Object.create(null)` for dictionaries
- Validate user input to prevent pollution
- Profile memory usage

Understanding prototypal inheritance is essential for:

- Debugging "is not a function" errors
- Optimizing memory usage
- Extending built-in objects safely
- Implementing design patterns
- Acing technical interviews
