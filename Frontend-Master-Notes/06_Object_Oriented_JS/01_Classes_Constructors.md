# Classes & Constructors

> **ES6 classes, constructor functions, and class features**

---

## 🎯 Overview

JavaScript supports both constructor functions (pre-ES6) and class syntax (ES6+). Understanding both approaches and their features is essential for modern JavaScript development.

---

## 📚 Core Concepts

### **Concept 1: Constructor Functions (Pre-ES6)**

```javascript
// Constructor function
function Person(name, age) {
  this.name = name;
  this.age = age;
}

// Methods on prototype
Person.prototype.greet = function () {
  return `Hello, I'm ${this.name}`;
};

Person.prototype.haveBirthday = function () {
  this.age++;
};

// Static method
Person.createAnonymous = function () {
  return new Person("Anonymous", 0);
};

// Usage
const person = new Person("Alice", 25);
console.log(person.greet()); // Hello, I'm Alice
person.haveBirthday();
console.log(person.age); // 26

const anon = Person.createAnonymous();
```

### **Concept 2: ES6 Classes**

```javascript
// ES6 class
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }

  // Instance method (on prototype)
  greet() {
    return `Hello, I'm ${this.name}`;
  }

  haveBirthday() {
    this.age++;
  }

  // Getter
  get info() {
    return `${this.name}, ${this.age} years old`;
  }

  // Setter
  set info(value) {
    const [name, age] = value.split(", ");
    this.name = name;
    this.age = parseInt(age);
  }

  // Static method (on class itself)
  static createAnonymous() {
    return new Person("Anonymous", 0);
  }

  // Static property
  static species = "Homo sapiens";
}

// Usage
const person = new Person("Alice", 25);
console.log(person.greet()); // Hello, I'm Alice
console.log(person.info); // Alice, 25 years old
person.info = "Bob, 30";
console.log(person.name); // Bob

const anon = Person.createAnonymous();
console.log(Person.species); // Homo sapiens
```

### **Concept 3: Class Features**

**Public Fields:**

```javascript
class Counter {
  count = 0; // Public field

  increment() {
    this.count++;
  }
}

const counter = new Counter();
console.log(counter.count); // 0
```

**Private Fields (# prefix):**

```javascript
class BankAccount {
  #balance = 0; // Private field

  deposit(amount) {
    this.#balance += amount;
  }

  getBalance() {
    return this.#balance;
  }
}

const account = new BankAccount();
account.deposit(100);
console.log(account.getBalance()); // 100
console.log(account.#balance); // SyntaxError
```

**Static Blocks:**

```javascript
class Config {
  static #apiKey;

  static {
    // Static initialization block
    this.#apiKey = process.env.API_KEY || "default";
  }

  static getApiKey() {
    return this.#apiKey;
  }
}
```

---

## 🔥 Practical Examples

### **Example 1: Real-World Class**

```javascript
class User {
  #password; // Private

  constructor(username, password) {
    this.username = username;
    this.#password = this.#hashPassword(password);
    this.createdAt = new Date();
    this.lastLogin = null;
  }

  #hashPassword(password) {
    // Simplified hash
    return btoa(password);
  }

  verifyPassword(password) {
    return this.#hashPassword(password) === this.#password;
  }

  login() {
    this.lastLogin = new Date();
    return true;
  }

  get accountAge() {
    return Math.floor((Date.now() - this.createdAt) / (1000 * 60 * 60 * 24));
  }

  static fromJSON(json) {
    const data = JSON.parse(json);
    return new User(data.username, data.password);
  }
}

const user = new User("alice", "secret123");
user.login();
console.log(user.accountAge); // Days since creation
console.log(user.verifyPassword("secret123")); // true
```

### **Example 2: Factory Pattern with Classes**

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    return "Some sound";
  }
}

class Dog extends Animal {
  speak() {
    return "Woof!";
  }
}

class Cat extends Animal {
  speak() {
    return "Meow!";
  }
}

class AnimalFactory {
  static create(type, name) {
    switch (type) {
      case "dog":
        return new Dog(name);
      case "cat":
        return new Cat(name);
      default:
        return new Animal(name);
    }
  }
}

const dog = AnimalFactory.create("dog", "Buddy");
console.log(dog.speak()); // Woof!
```

### **Example 3: Singleton Pattern**

```javascript
class Database {
  static #instance = null;
  #connection = null;

  constructor() {
    if (Database.#instance) {
      return Database.#instance;
    }
    Database.#instance = this;
  }

  connect() {
    if (!this.#connection) {
      this.#connection = "Connected";
      console.log("Database connected");
    }
    return this.#connection;
  }

  static getInstance() {
    if (!Database.#instance) {
      Database.#instance = new Database();
    }
    return Database.#instance;
  }
}

// All reference same instance
const db1 = Database.getInstance();
const db2 = Database.getInstance();
const db3 = new Database();

console.log(db1 === db2); // true
console.log(db2 === db3); // true
```

### **Example 4: Builder Pattern**

```javascript
class Query {
  #table;
  #where = [];
  #select = "*";
  #limit;

  from(table) {
    this.#table = table;
    return this;
  }

  where(condition) {
    this.#where.push(condition);
    return this;
  }

  select(fields) {
    this.#select = fields;
    return this;
  }

  limit(n) {
    this.#limit = n;
    return this;
  }

  build() {
    let sql = `SELECT ${this.#select} FROM ${this.#table}`;

    if (this.#where.length > 0) {
      sql += ` WHERE ${this.#where.join(" AND ")}`;
    }

    if (this.#limit) {
      sql += ` LIMIT ${this.#limit}`;
    }

    return sql;
  }
}

const query = new Query()
  .from("users")
  .select("name, email")
  .where("age > 18")
  .where("active = true")
  .limit(10)
  .build();

console.log(query);
// SELECT name, email FROM users WHERE age > 18 AND active = true LIMIT 10
```

---

## 🏗️ Best Practices

1. **Use ES6 classes** - More readable and standard

   ```javascript
   class User {
     /* ... */
   }
   ```

2. **Use private fields for encapsulation** - Hide implementation details

   ```javascript
   class Account {
     #balance = 0;
   }
   ```

3. **Initialize all properties in constructor** - For V8 optimization

   ```javascript
   constructor() {
     this.prop1 = null;
     this.prop2 = null;
   }
   ```

4. **Use static methods for utilities** - Don't need instance

   ```javascript
   static fromJSON(json) { /* ... */ }
   ```

5. **Prefer composition over inheritance** - More flexible
   ```javascript
   class User {
     constructor() {
       this.notifier = new Notifier();
     }
   }
   ```

---

## 🧪 Interview Questions

### **Q1: What's the difference between ES6 classes and constructor functions?**

**Answer:** ES6 classes are syntactic sugar over constructor functions but with differences:

- Classes must be called with `new`
- Class methods are non-enumerable
- Classes have `static` keyword
- Classes support `extends` for inheritance
- Classes can have private fields (#)

Both create objects with prototypal inheritance.

### **Q2: What are private fields in JavaScript?**

**Answer:** Private fields use `#` prefix and are truly private (not accessible outside class):

```javascript
class Example {
  #private = "secret";
  public = "visible";

  getPrivate() {
    return this.#private;
  }
}

const obj = new Example();
obj.public; // 'visible'
obj.#private; // SyntaxError
```

### **Q3: How do you create a singleton in JavaScript?**

**Answer:** Use a static instance property:

```javascript
class Singleton {
  static #instance;

  static getInstance() {
    if (!Singleton.#instance) {
      Singleton.#instance = new Singleton();
    }
    return Singleton.#instance;
  }
}
```

---

## 📚 Key Takeaways

- ES6 classes are syntactic sugar over constructor functions
- Use classes for OOP patterns - cleaner syntax
- Private fields (#) provide true encapsulation
- Static methods/properties belong to class, not instances
- Initialize all properties in constructor for optimization
- Getters/setters provide controlled property access
- Classes support modern patterns: Factory, Singleton, Builder

---

**Master classes & constructors for production-ready code!**
