# JavaScript Fundamentals

> **Master the core concepts that every senior JavaScript developer must know**

---

## 🎯 Overview

This section covers the fundamental concepts of JavaScript that form the foundation of everything else. These topics are essential for understanding how JavaScript works under the hood and are frequently asked in technical interviews.

---

## 📚 Topics Covered

### **✅ 01. Execution Context & Scope**

- Global vs Function vs Block Execution Context
- Creation Phase vs Execution Phase
- Scope Chain and Lexical Environment
- `this` binding rules

### **✅ 02. Hoisting & Temporal Dead Zone (TDZ)**

- Variable hoisting with `var`, `let`, `const`
- Function hoisting vs function expressions
- Temporal Dead Zone for `let` and `const`
- Best practices to avoid hoisting issues

### **✅ 03. Closures & Scope Chain**

- What are closures and how do they work
- Lexical scoping and closure creation
- Memory implications of closures
- Practical use cases: data privacy, callbacks, currying

### **✅ 04. Prototypal Inheritance**

- Prototype chain mechanism
- `__proto__` vs `prototype`
- `Object.create()` and inheritance patterns
- Modern class syntax vs prototypal inheritance

### **📝 05. This Binding**

- Default, implicit, explicit, and `new` binding
- Arrow functions and lexical `this`
- `call()`, `apply()`, and `bind()`
- Common pitfalls and solutions

### **✅ 06. Event Loop & Microtasks**

- Call stack and execution model
- Task queue (macro tasks) vs Microtask queue
- How promises fit into the event loop
- `setTimeout`, `setImmediate`, `process.nextTick`

### **📝 07. Type Coercion & Type Conversion**

- Implicit vs explicit coercion
- `==` vs `===` comparison
- String, number, and boolean coercion
- Common pitfalls and best practices

---

## 🎓 Learning Path

### **For Beginners**

Start here if you're new to JavaScript or need to solidify your fundamentals:

1. **Execution Context & Scope** - Understand how code is executed
2. **Hoisting & TDZ** - Learn variable and function declarations
3. **This Binding** - Master the `this` keyword
4. **Type Coercion** - Understand implicit and explicit conversions

### **For Intermediate Developers**

If you already know the basics, focus on these advanced topics:

1. **Closures & Scope Chain** - Deep dive into closure mechanics
2. **Prototypal Inheritance** - Understand JavaScript's inheritance model
3. **Event Loop & Microtasks** - Master asynchronous JavaScript

### **For Interview Preparation**

Focus on these high-frequency interview topics:

1. **Closures** - Most common JavaScript interview question
2. **Event Loop** - Understanding async behavior
3. **Prototypal Inheritance** - OOP in JavaScript
4. **This Binding** - Context and scope questions

---

## 🎯 Key Concepts to Master

### **1. Understanding Execution Context**

Every JavaScript code runs inside an execution context:

```javascript
// Global Execution Context
var globalVar = "global";

function outer() {
  // Function Execution Context for outer()
  var outerVar = "outer";

  function inner() {
    // Function Execution Context for inner()
    var innerVar = "inner";
    console.log(globalVar, outerVar, innerVar);
  }

  inner();
}

outer();
```

**Key Points:**

- **Global Context** - Created when the script starts
- **Function Context** - Created when a function is invoked
- **Block Context** - Created with `let`/`const` in blocks

---

### **2. Mastering Closures**

Closures are functions that remember their lexical scope:

```javascript
function createCounter() {
  let count = 0;

  return {
    increment() {
      return ++count;
    },
    decrement() {
      return --count;
    },
    getCount() {
      return count;
    },
  };
}

const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
console.log(counter.getCount()); // 2
```

**Key Points:**

- Functions "close over" their lexical environment
- Enable data privacy and encapsulation
- Critical for callbacks, event handlers, and higher-order functions

---

### **3. Understanding Prototypes**

JavaScript uses prototypal inheritance:

```javascript
function Animal(name) {
  this.name = name;
}

Animal.prototype.speak = function () {
  console.log(`${this.name} makes a sound`);
};

const dog = new Animal("Dog");
dog.speak(); // "Dog makes a sound"

console.log(dog.__proto__ === Animal.prototype); // true
```

**Key Points:**

- Every object has a hidden `[[Prototype]]` property
- Prototype chain enables inheritance
- Modern classes are syntactic sugar over prototypes

---

### **4. Event Loop Mastery**

Understanding the event loop is crucial for async programming:

```javascript
console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve().then(() => console.log("3"));

console.log("4");

// Output: 1, 4, 3, 2
```

**Key Points:**

- Call stack executes synchronously
- Microtasks (promises) have higher priority than macrotasks (setTimeout)
- Understanding this is essential for async code

---

## 🔥 Common Interview Questions

### **Closures**

- What is a closure? Can you provide an example?
- Explain how closures can lead to memory leaks
- How would you implement data privacy using closures?

### **Event Loop**

- Explain the JavaScript event loop
- What's the difference between microtasks and macrotasks?
- Why does Promise.then execute before setTimeout?

### **This Binding**

- What are the four rules of `this` binding?
- How do arrow functions handle `this`?
- What's the difference between `call`, `apply`, and `bind`?

### **Prototypes**

- How does prototypal inheritance work?
- What's the difference between `__proto__` and `prototype`?
- How do ES6 classes relate to prototypes?

---

## 📝 Practical Exercises

### **Exercise 1: Closure Practice**

Implement a function that creates a private counter:

```javascript
function createSecureCounter(initialValue) {
  // Your code here
  // Should have increment, decrement, and reset methods
  // Count should not be directly accessible
}
```

### **Exercise 2: Prototype Chain**

Create an inheritance hierarchy:

```javascript
// Implement Vehicle -> Car -> ElectricCar
// Each should add specific properties/methods
```

### **Exercise 3: Event Loop**

Predict the output:

```javascript
async function async1() {
  console.log("async1 start");
  await async2();
  console.log("async1 end");
}

async function async2() {
  console.log("async2");
}

console.log("script start");

setTimeout(() => {
  console.log("setTimeout");
}, 0);

async1();

new Promise((resolve) => {
  console.log("promise1");
  resolve();
}).then(() => {
  console.log("promise2");
});

console.log("script end");
```

---

## 🎯 Best Practices

### **1. Variable Declarations**

- Use `const` by default
- Use `let` when reassignment is needed
- Avoid `var` to prevent hoisting issues

### **2. Function Declarations**

- Use arrow functions for callbacks to preserve `this`
- Use regular functions for methods that need `this` binding
- Understand when to use function declarations vs expressions

### **3. Async Code**

- Prefer `async/await` over raw promises for readability
- Understand the event loop to debug async issues
- Always handle promise rejections

### **4. Prototypes**

- Use ES6 classes for clearer syntax
- Understand that classes are syntactic sugar
- Know when to use `Object.create()` for more control

---

## 📚 Additional Resources

### **MDN Documentation**

- [Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures)
- [Event Loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)
- [Inheritance and the prototype chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)

### **Books**

- _You Don't Know JS_ by Kyle Simpson
- _JavaScript: The Good Parts_ by Douglas Crockford
- _Eloquent JavaScript_ by Marijn Haverbeke

### **Videos**

- Philip Roberts: "What the heck is the event loop anyway?"
- Kyle Simpson: "Advanced JavaScript" course

---

## 🎓 Key Takeaways

1. **Execution Context** determines how code runs and variables are accessed
2. **Closures** enable powerful patterns like data privacy and memoization
3. **Prototypal Inheritance** is how JavaScript implements object-oriented programming
4. **This Binding** follows four rules: default, implicit, explicit, new
5. **Event Loop** coordinates synchronous and asynchronous code execution
6. **Type Coercion** can cause subtle bugs - use `===` and be explicit

---

## ✅ Completion Checklist

- [ ] Understand execution contexts and scope
- [ ] Can explain closures with real examples
- [ ] Know the prototype chain mechanism
- [ ] Master the four `this` binding rules
- [ ] Understand the event loop and microtasks
- [ ] Can predict type coercion behavior
- [ ] Comfortable with hoisting rules
- [ ] Can explain TDZ for `let` and `const`

---

**Next Steps:** Once you've mastered these fundamentals, move on to **Advanced JavaScript** topics like generators, proxies, and async iterators!

---

_Last updated: January 2026_
