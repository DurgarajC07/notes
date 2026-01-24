# `this` Binding in JavaScript

> **Understanding how `this` is determined in different contexts**

---

## 🎯 What is `this`?

`this` is a special keyword in JavaScript that refers to the context in which a function is executed. Unlike other programming languages where `this` always refers to the instance of a class, JavaScript's `this` is dynamically determined at **call-time**, not **write-time**.

---

## 📚 The Four Binding Rules

### **1. Default Binding**

When a function is called in non-strict mode without any context, `this` defaults to the global object.

```javascript
function showThis() {
  console.log(this); // window (browser) or global (Node.js)
}

showThis(); // Default binding
```

**In strict mode:**

```javascript
"use strict";

function showThis() {
  console.log(this); // undefined
}

showThis();
```

---

### **2. Implicit Binding**

When a function is called as a method of an object, `this` refers to that object.

```javascript
const user = {
  name: "Alice",
  greet() {
    console.log(`Hello, I'm ${this.name}`);
  },
};

user.greet(); // "Hello, I'm Alice" - this = user
```

**⚠️ Losing Implicit Binding:**

```javascript
const user = {
  name: "Alice",
  greet() {
    console.log(`Hello, I'm ${this.name}`);
  },
};

const greetFn = user.greet;
greetFn(); // "Hello, I'm undefined" - this = window/global
```

**Why?** The function is called without a context, so default binding applies.

---

### **3. Explicit Binding**

You can explicitly set `this` using `call()`, `apply()`, or `bind()`.

#### **`call()`**

```javascript
function greet(greeting, punctuation) {
  console.log(`${greeting}, I'm ${this.name}${punctuation}`);
}

const user = { name: "Bob" };

greet.call(user, "Hello", "!"); // "Hello, I'm Bob!"
```

#### **`apply()`**

Same as `call()`, but takes arguments as an array:

```javascript
greet.apply(user, ["Hi", "."]); // "Hi, I'm Bob."
```

#### **`bind()`**

Returns a new function with `this` permanently bound:

```javascript
const boundGreet = greet.bind(user, "Hey");
boundGreet("!!!"); // "Hey, I'm Bob!!!"
```

**Hard Binding Pattern:**

```javascript
function hardBind(fn, context) {
  return function (...args) {
    return fn.apply(context, args);
  };
}

const user = { name: "Charlie" };
const sayHi = hardBind(function () {
  console.log(this.name);
}, user);

sayHi(); // "Charlie"
```

---

### **4. `new` Binding**

When a function is called with `new`, a new object is created and `this` is bound to it.

```javascript
function User(name) {
  this.name = name;
  this.greet = function () {
    console.log(`Hi, I'm ${this.name}`);
  };
}

const alice = new User("Alice");
alice.greet(); // "Hi, I'm Alice" - this = alice
```

**What happens with `new`:**

1. A new empty object is created
2. The object's prototype is set
3. `this` is bound to the new object
4. The constructor function is executed
5. The new object is returned (unless the constructor returns an object)

---

## 🔥 Arrow Functions

Arrow functions **do not have their own `this`**. They lexically inherit `this` from the enclosing scope.

```javascript
const user = {
  name: "Diana",
  greet: function () {
    setTimeout(function () {
      console.log(this.name); // undefined - this = window
    }, 100);
  },
};

user.greet();
```

**Fixed with arrow function:**

```javascript
const user = {
  name: "Diana",
  greet: function () {
    setTimeout(() => {
      console.log(this.name); // "Diana" - this inherited from greet()
    }, 100);
  },
};

user.greet();
```

**⚠️ Arrow functions cannot be bound:**

```javascript
const greet = () => {
  console.log(this);
};

const user = { name: "Eve" };
greet.call(user); // Still window/global - cannot change this
```

---

## 📊 Binding Precedence

When multiple rules apply, this is the order of precedence:

1. **`new` binding** - highest priority
2. **Explicit binding** (`call`, `apply`, `bind`)
3. **Implicit binding** (method call)
4. **Default binding** - lowest priority

```javascript
function greet() {
  console.log(this.name);
}

const user1 = { name: "User1" };
const user2 = { name: "User2" };

// Implicit vs Explicit
const boundGreet = greet.bind(user1);
user2.greet = boundGreet;
user2.greet(); // "User1" - explicit binding wins

// Explicit vs new
function User(name) {
  this.name = name;
}

const boundUser = User.bind({ name: "Ignored" });
const instance = new boundUser("Frank");
console.log(instance.name); // "Frank" - new binding wins
```

---

## 🎯 Common Pitfalls

### **1. Callback Functions**

```javascript
const user = {
  name: "Grace",
  hobbies: ["reading", "coding"],
  showHobbies() {
    this.hobbies.forEach(function (hobby) {
      console.log(`${this.name} likes ${hobby}`);
      // this = window/undefined
    });
  },
};

user.showHobbies(); // "undefined likes reading"
```

**Solutions:**

```javascript
// Solution 1: Arrow function
showHobbies() {
  this.hobbies.forEach(hobby => {
    console.log(`${this.name} likes ${hobby}`);
  });
}

// Solution 2: Pass thisArg to forEach
showHobbies() {
  this.hobbies.forEach(function(hobby) {
    console.log(`${this.name} likes ${hobby}`);
  }, this); // thisArg parameter
}

// Solution 3: Store this reference
showHobbies() {
  const self = this;
  this.hobbies.forEach(function(hobby) {
    console.log(`${self.name} likes ${hobby}`);
  });
}
```

---

### **2. Event Handlers**

```javascript
class Button {
  constructor(label) {
    this.label = label;
  }

  handleClick() {
    console.log(`Button ${this.label} clicked`);
  }

  render() {
    const button = document.createElement("button");
    button.addEventListener("click", this.handleClick);
    // this.handleClick loses context
  }
}
```

**Solutions:**

```javascript
// Solution 1: Bind in constructor
constructor(label) {
  this.label = label;
  this.handleClick = this.handleClick.bind(this);
}

// Solution 2: Arrow function
handleClick = () => {
  console.log(`Button ${this.label} clicked`);
}

// Solution 3: Inline arrow function
button.addEventListener('click', () => this.handleClick());
```

---

### **3. Method Assignment**

```javascript
const calculator = {
  value: 0,
  add(n) {
    this.value += n;
    return this;
  },
};

const add = calculator.add;
add(5); // Error: Cannot read property 'value' of undefined
```

**Solution:**

```javascript
const add = calculator.add.bind(calculator);
add(5); // Works correctly
```

---

## 🏗️ Real-World Patterns

### **1. React Class Components**

```javascript
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };

    // Bind methods in constructor
    this.increment = this.increment.bind(this);
  }

  increment() {
    this.setState({ count: this.state.count + 1 });
  }

  render() {
    return <button onClick={this.increment}>Count: {this.state.count}</button>;
  }
}
```

**Modern approach with class fields:**

```javascript
class Counter extends React.Component {
  state = { count: 0 };

  increment = () => {
    this.setState({ count: this.state.count + 1 });
  };

  render() {
    return <button onClick={this.increment}>Count: {this.state.count}</button>;
  }
}
```

---

### **2. Method Chaining**

```javascript
class QueryBuilder {
  constructor() {
    this.query = "";
  }

  select(fields) {
    this.query += `SELECT ${fields} `;
    return this; // Return this for chaining
  }

  from(table) {
    this.query += `FROM ${table} `;
    return this;
  }

  where(condition) {
    this.query += `WHERE ${condition}`;
    return this;
  }

  build() {
    return this.query;
  }
}

const query = new QueryBuilder()
  .select("*")
  .from("users")
  .where("age > 18")
  .build();
```

---

### **3. Partial Application**

```javascript
function multiply(a, b) {
  return a * b;
}

const double = multiply.bind(null, 2);
const triple = multiply.bind(null, 3);

console.log(double(5)); // 10
console.log(triple(5)); // 15
```

---

## 🧪 Interview Questions

### **Q1: What will this output?**

```javascript
var name = "Global";

const obj = {
  name: "Object",
  getName: function () {
    return this.name;
  },
};

const getName = obj.getName;
console.log(getName());
```

**Answer:** `"Global"` - The function loses its context when assigned to a variable.

---

### **Q2: Fix the following code:**

```javascript
const user = {
  name: "John",
  friends: ["Jane", "Jack"],
  greetFriends() {
    this.friends.forEach(function (friend) {
      console.log(`${this.name} greets ${friend}`);
    });
  },
};
```

**Solution:**

```javascript
greetFriends() {
  this.friends.forEach(friend => {
    console.log(`${this.name} greets ${friend}`);
  });
}
```

---

### **Q3: What's the difference between `call`, `apply`, and `bind`?**

- **`call()`**: Invokes immediately with individual arguments
- **`apply()`**: Invokes immediately with arguments as an array
- **`bind()`**: Returns a new function with `this` bound (doesn't invoke immediately)

---

## 🎯 Best Practices

1. **Use arrow functions for callbacks** to preserve lexical `this`
2. **Bind event handlers in constructor** for React class components
3. **Avoid storing `this` in variables** like `const self = this` (use arrow functions instead)
4. **Use explicit binding** when passing methods as callbacks
5. **Understand the call-site** - where the function is called determines `this`
6. **Prefer arrow functions** in class fields for auto-binding in modern JavaScript
7. **Use strict mode** to catch accidental global `this` binding

---

## 📚 Key Takeaways

- `this` is determined by **how a function is called**, not where it's defined
- **Arrow functions** inherit `this` lexically from their enclosing scope
- **Four binding rules**: default, implicit, explicit, `new`
- **Precedence**: `new` > explicit > implicit > default
- Common pitfalls: callbacks, event handlers, method assignment
- Modern solution: Use arrow functions for lexical `this` binding

---

**Practice Exercise:** Identify the value of `this` in different scenarios and understand why!
