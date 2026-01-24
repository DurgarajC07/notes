# Private Fields

> **Encapsulation strategies**

---

## 🎯 Overview

Encapsulation hides internal implementation details from outside access. JavaScript provides multiple ways to achieve privacy: private class fields (#), WeakMaps, and closures.

---

## 📚 Core Concepts

### **Concept 1: Private Class Fields (#)**

ES2022 introduced true private fields using `#` prefix:

```javascript
class BankAccount {
  // Private fields
  #balance = 0;
  #accountNumber;

  // Public fields
  owner;

  constructor(owner, accountNumber, initialBalance) {
    this.owner = owner;
    this.#accountNumber = accountNumber;
    this.#balance = initialBalance;
  }

  // Private method
  #validateAmount(amount) {
    return amount > 0 && amount <= this.#balance;
  }

  // Public methods
  deposit(amount) {
    if (amount > 0) {
      this.#balance += amount;
      return true;
    }
    return false;
  }

  withdraw(amount) {
    if (this.#validateAmount(amount)) {
      this.#balance -= amount;
      return true;
    }
    return false;
  }

  getBalance() {
    return this.#balance;
  }

  // Getter for private field
  get accountInfo() {
    return {
      owner: this.owner,
      account: this.#accountNumber,
      balance: this.#balance,
    };
  }
}

const account = new BankAccount("Alice", "ACC123", 1000);
account.deposit(500);
console.log(account.getBalance()); // 1500

// Cannot access private fields
console.log(account.#balance); // SyntaxError
console.log(account.#validateAmount); // SyntaxError
```

**Private Static Fields:**

```javascript
class Counter {
  static #instanceCount = 0;
  #id;

  constructor() {
    this.#id = ++Counter.#instanceCount;
  }

  getId() {
    return this.#id;
  }

  static getInstanceCount() {
    return Counter.#instanceCount;
  }
}

const c1 = new Counter();
const c2 = new Counter();

console.log(c1.getId()); // 1
console.log(c2.getId()); // 2
console.log(Counter.getInstanceCount()); // 2
```

### **Concept 2: WeakMap Pattern (Pre-# syntax)**

Before private fields, WeakMap was used for privacy:

```javascript
const privateData = new WeakMap();

class Person {
  constructor(name, age) {
    // Store private data
    privateData.set(this, {
      name,
      age,
      secrets: [],
    });
  }

  getName() {
    return privateData.get(this).name;
  }

  getAge() {
    return privateData.get(this).age;
  }

  addSecret(secret) {
    privateData.get(this).secrets.push(secret);
  }

  getSecrets() {
    return [...privateData.get(this).secrets];
  }
}

const person = new Person("Alice", 25);
person.addSecret("Loves chocolate");

console.log(person.getName()); // Alice
console.log(person.getSecrets()); // ['Loves chocolate']

// Cannot access private data
console.log(person.name); // undefined
console.log(person.secrets); // undefined
```

**Benefits of WeakMap:**

- True privacy (not accessible outside)
- Automatic garbage collection
- No memory leaks

### **Concept 3: Closure-Based Privacy**

Constructor functions with closures:

```javascript
function createCounter() {
  // Private variable
  let count = 0;

  // Private function
  function logCount() {
    console.log(`Count: ${count}`);
  }

  // Return public API
  return {
    increment() {
      count++;
      logCount();
    },

    decrement() {
      count--;
      logCount();
    },

    getCount() {
      return count;
    },
  };
}

const counter = createCounter();
counter.increment(); // Count: 1
counter.increment(); // Count: 2
console.log(counter.getCount()); // 2

// Cannot access private
console.log(counter.count); // undefined
```

**Module Pattern:**

```javascript
const Calculator = (function () {
  // Private
  let history = [];

  function addToHistory(operation, result) {
    history.push({ operation, result, timestamp: Date.now() });
  }

  // Public API
  return {
    add(a, b) {
      const result = a + b;
      addToHistory(`${a} + ${b}`, result);
      return result;
    },

    subtract(a, b) {
      const result = a - b;
      addToHistory(`${a} - ${b}`, result);
      return result;
    },

    getHistory() {
      return [...history]; // Return copy
    },

    clearHistory() {
      history = [];
    },
  };
})();

Calculator.add(5, 3); // 8
Calculator.subtract(10, 4); // 6
console.log(Calculator.getHistory());
// [{ operation: '5 + 3', result: 8, ... }, ...]
```

---

## 🔥 Practical Examples

### **Example 1: Secure User Class**

```javascript
class User {
  #password;
  #loginAttempts = 0;
  #locked = false;

  static #MAX_ATTEMPTS = 3;

  constructor(username, password) {
    this.username = username;
    this.#password = this.#hashPassword(password);
  }

  #hashPassword(password) {
    // Simplified hash
    return btoa(password);
  }

  #isLocked() {
    return this.#locked;
  }

  #resetAttempts() {
    this.#loginAttempts = 0;
    this.#locked = false;
  }

  login(password) {
    if (this.#isLocked()) {
      return { success: false, message: "Account locked" };
    }

    if (this.#hashPassword(password) === this.#password) {
      this.#resetAttempts();
      return { success: true, message: "Login successful" };
    }

    this.#loginAttempts++;

    if (this.#loginAttempts >= User.#MAX_ATTEMPTS) {
      this.#locked = true;
      return { success: false, message: "Account locked" };
    }

    return {
      success: false,
      message: `Invalid password. ${User.#MAX_ATTEMPTS - this.#loginAttempts} attempts remaining`,
    };
  }

  changePassword(oldPassword, newPassword) {
    if (this.#hashPassword(oldPassword) !== this.#password) {
      return false;
    }
    this.#password = this.#hashPassword(newPassword);
    return true;
  }
}

const user = new User("alice", "secret123");
console.log(user.login("wrong")); // Invalid password...
console.log(user.login("secret123")); // Login successful
```

### **Example 2: Observable with Private State**

```javascript
class Observable {
  #observers = new Set();
  #value;

  constructor(initialValue) {
    this.#value = initialValue;
  }

  get value() {
    return this.#value;
  }

  set value(newValue) {
    if (newValue !== this.#value) {
      const oldValue = this.#value;
      this.#value = newValue;
      this.#notify(newValue, oldValue);
    }
  }

  #notify(newValue, oldValue) {
    this.#observers.forEach((observer) => {
      observer(newValue, oldValue);
    });
  }

  subscribe(observer) {
    this.#observers.add(observer);
    return () => this.#observers.delete(observer);
  }
}

const count = new Observable(0);

const unsubscribe = count.subscribe((newVal, oldVal) => {
  console.log(`Count changed from ${oldVal} to ${newVal}`);
});

count.value = 1; // Count changed from 0 to 1
count.value = 2; // Count changed from 1 to 2

unsubscribe();
count.value = 3; // No output (unsubscribed)
```

---

## 🏗️ Best Practices

1. **Use private fields (#) for new code** - Modern and clean
2. **Provide getters/setters for controlled access** - Validation and side effects
3. **WeakMap for true privacy in older environments** - Backwards compatibility
4. **Don't expose private data in return values** - Return copies, not references
5. **Use private methods for internal logic** - Hide implementation details

---

## 🧪 Interview Questions

### **Q1: How do private fields work in JavaScript?**

**Answer:** Private fields use `#` prefix and are truly private - not accessible outside the class. They're defined in the class body and can be accessed only within class methods.

```javascript
class Example {
  #private = "secret";

  getPrivate() {
    return this.#private;
  }
}

const obj = new Example();
obj.#private; // SyntaxError
```

### **Q2: What's the difference between private fields and WeakMap?**

**Answer:**

- **Private fields (#)**: Newer, cleaner syntax, built-in language feature
- **WeakMap**: Older pattern, works in all environments, more verbose

Both provide true privacy. Use private fields for modern projects.

---

## 📚 Key Takeaways

- Private fields (#) provide true encapsulation in ES2022+
- Use private methods for internal implementation details
- WeakMap pattern for privacy in older environments
- Closures enable privacy in functional patterns
- Don't expose private data directly in return values
- Private fields not inherited by subclasses
- Encapsulation improves maintainability and security

---

**Master private fields for production-ready code!**
