# Design Patterns

> **Gang of Four patterns**

---

## 🎯 Overview

Design patterns are proven solutions to common problems in software design. Understanding these patterns helps you write maintainable, scalable, and testable code.

---

## 📚 Core Concepts

### **Concept 1: Creational Patterns**

**Singleton Pattern:**

```javascript
class Singleton {
  static #instance;

  constructor() {
    if (Singleton.#instance) {
      return Singleton.#instance;
    }
    Singleton.#instance = this;
    this.data = [];
  }

  static getInstance() {
    if (!Singleton.#instance) {
      Singleton.#instance = new Singleton();
    }
    return Singleton.#instance;
  }
}

const s1 = Singleton.getInstance();
const s2 = Singleton.getInstance();
console.log(s1 === s2); // true
```

**Factory Pattern:**

```javascript
class Button {
  render() {}
}

class WindowsButton extends Button {
  render() {
    return "<button>Windows Button</button>";
  }
}

class MacButton extends Button {
  render() {
    return "<button>Mac Button</button>";
  }
}

class ButtonFactory {
  static create(os) {
    switch (os) {
      case "windows":
        return new WindowsButton();
      case "mac":
        return new MacButton();
      default:
        throw new Error("Unknown OS");
    }
  }
}

const button = ButtonFactory.create("windows");
console.log(button.render());
```

**Builder Pattern:**

```javascript
class QueryBuilder {
  constructor() {
    this._query = {};
  }

  select(fields) {
    this._query.select = fields;
    return this;
  }

  from(table) {
    this._query.from = table;
    return this;
  }

  where(condition) {
    this._query.where = this._query.where || [];
    this._query.where.push(condition);
    return this;
  }

  limit(count) {
    this._query.limit = count;
    return this;
  }

  build() {
    let sql = `SELECT ${this._query.select || "*"}`;
    sql += ` FROM ${this._query.from}`;

    if (this._query.where) {
      sql += ` WHERE ${this._query.where.join(" AND ")}`;
    }

    if (this._query.limit) {
      sql += ` LIMIT ${this._query.limit}`;
    }

    return sql;
  }
}

const query = new QueryBuilder()
  .select("name, email")
  .from("users")
  .where("age > 18")
  .where("active = true")
  .limit(10)
  .build();

console.log(query);
// SELECT name, email FROM users WHERE age > 18 AND active = true LIMIT 10
```

### **Concept 2: Structural Patterns**

**Module Pattern:**

```javascript
const CounterModule = (function () {
  // Private
  let count = 0;

  function logCount() {
    console.log(`Count: ${count}`);
  }

  // Public API
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
})();

CounterModule.increment(); // Count: 1
CounterModule.increment(); // Count: 2
console.log(CounterModule.getCount()); // 2
```

**Decorator Pattern:**

```javascript
class Coffee {
  cost() {
    return 5;
  }

  description() {
    return "Coffee";
  }
}

class CoffeeDecorator {
  constructor(coffee) {
    this.coffee = coffee;
  }

  cost() {
    return this.coffee.cost();
  }

  description() {
    return this.coffee.description();
  }
}

class Milk extends CoffeeDecorator {
  cost() {
    return this.coffee.cost() + 1;
  }

  description() {
    return this.coffee.description() + ", Milk";
  }
}

class Sugar extends CoffeeDecorator {
  cost() {
    return this.coffee.cost() + 0.5;
  }

  description() {
    return this.coffee.description() + ", Sugar";
  }
}

let coffee = new Coffee();
coffee = new Milk(coffee);
coffee = new Sugar(coffee);

console.log(coffee.description()); // Coffee, Milk, Sugar
console.log(coffee.cost()); // 6.5
```

**Proxy Pattern:**

```javascript
class RealSubject {
  request() {
    console.log("RealSubject: Handling request");
  }
}

class ProxySubject {
  constructor() {
    this.realSubject = new RealSubject();
  }

  request() {
    if (this.checkAccess()) {
      this.realSubject.request();
      this.logAccess();
    }
  }

  checkAccess() {
    console.log("Proxy: Checking access");
    return true;
  }

  logAccess() {
    console.log("Proxy: Logging access");
  }
}

const proxy = new ProxySubject();
proxy.request();
// Proxy: Checking access
// RealSubject: Handling request
// Proxy: Logging access
```

### **Concept 3: Behavioral Patterns**

**Observer Pattern:**

```javascript
class Subject {
  constructor() {
    this.observers = [];
  }

  subscribe(observer) {
    this.observers.push(observer);
  }

  unsubscribe(observer) {
    this.observers = this.observers.filter((obs) => obs !== observer);
  }

  notify(data) {
    this.observers.forEach((observer) => observer.update(data));
  }
}

class Observer {
  constructor(name) {
    this.name = name;
  }

  update(data) {
    console.log(`${this.name} received: ${data}`);
  }
}

const subject = new Subject();
const observer1 = new Observer("Observer 1");
const observer2 = new Observer("Observer 2");

subject.subscribe(observer1);
subject.subscribe(observer2);

subject.notify("Hello!");
// Observer 1 received: Hello!
// Observer 2 received: Hello!
```

**Strategy Pattern:**

```javascript
class PaymentStrategy {
  pay(amount) {}
}

class CreditCardStrategy extends PaymentStrategy {
  constructor(cardNumber) {
    super();
    this.cardNumber = cardNumber;
  }

  pay(amount) {
    console.log(`Paid ${amount} using Credit Card ${this.cardNumber}`);
  }
}

class PayPalStrategy extends PaymentStrategy {
  constructor(email) {
    super();
    this.email = email;
  }

  pay(amount) {
    console.log(`Paid ${amount} using PayPal ${this.email}`);
  }
}

class ShoppingCart {
  constructor(paymentStrategy) {
    this.paymentStrategy = paymentStrategy;
  }

  setPaymentStrategy(strategy) {
    this.paymentStrategy = strategy;
  }

  checkout(amount) {
    this.paymentStrategy.pay(amount);
  }
}

const cart = new ShoppingCart(new CreditCardStrategy("1234-5678"));
cart.checkout(100); // Paid 100 using Credit Card 1234-5678

cart.setPaymentStrategy(new PayPalStrategy("user@email.com"));
cart.checkout(50); // Paid 50 using PayPal user@email.com
```

**Command Pattern:**

```javascript
class Command {
  execute() {}
  undo() {}
}

class AddCommand extends Command {
  constructor(receiver, value) {
    super();
    this.receiver = receiver;
    this.value = value;
  }

  execute() {
    this.receiver.add(this.value);
  }

  undo() {
    this.receiver.subtract(this.value);
  }
}

class Calculator {
  constructor() {
    this.value = 0;
  }

  add(value) {
    this.value += value;
  }

  subtract(value) {
    this.value -= value;
  }

  getValue() {
    return this.value;
  }
}

class CommandHistory {
  constructor() {
    this.commands = [];
  }

  execute(command) {
    command.execute();
    this.commands.push(command);
  }

  undo() {
    const command = this.commands.pop();
    if (command) {
      command.undo();
    }
  }
}

const calc = new Calculator();
const history = new CommandHistory();

history.execute(new AddCommand(calc, 10));
console.log(calc.getValue()); // 10

history.execute(new AddCommand(calc, 5));
console.log(calc.getValue()); // 15

history.undo();
console.log(calc.getValue()); // 10
```

---

## 🔥 Practical Examples

### **Example 1: Event Emitter (Observer Pattern)**

```javascript
class EventEmitter {
  constructor() {
    this.events = {};
  }

  on(event, callback) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(callback);
  }

  off(event, callback) {
    if (!this.events[event]) return;
    this.events[event] = this.events[event].filter((cb) => cb !== callback);
  }

  emit(event, data) {
    if (!this.events[event]) return;
    this.events[event].forEach((callback) => callback(data));
  }

  once(event, callback) {
    const wrapper = (data) => {
      callback(data);
      this.off(event, wrapper);
    };
    this.on(event, wrapper);
  }
}

const emitter = new EventEmitter();

emitter.on("user:login", (user) => {
  console.log(`User logged in: ${user.name}`);
});

emitter.emit("user:login", { name: "Alice" });
// User logged in: Alice
```

### **Example 2: Middleware Pattern (Chain of Responsibility)**

```javascript
class Middleware {
  constructor() {
    this.middlewares = [];
  }

  use(fn) {
    this.middlewares.push(fn);
    return this;
  }

  execute(context) {
    let index = 0;

    const next = () => {
      if (index >= this.middlewares.length) return;
      const middleware = this.middlewares[index++];
      middleware(context, next);
    };

    next();
  }
}

const app = new Middleware();

app.use((ctx, next) => {
  console.log("Middleware 1: Before");
  next();
  console.log("Middleware 1: After");
});

app.use((ctx, next) => {
  console.log("Middleware 2: Before");
  ctx.data = "Modified";
  next();
  console.log("Middleware 2: After");
});

app.use((ctx, next) => {
  console.log("Middleware 3: Handling");
  console.log("Context:", ctx);
});

app.execute({ data: "Initial" });
// Middleware 1: Before
// Middleware 2: Before
// Middleware 3: Handling
// Context: { data: 'Modified' }
// Middleware 2: After
// Middleware 1: After
```

### **Example 3: Repository Pattern**

```javascript
class Repository {
  constructor(storage) {
    this.storage = storage;
  }

  findById(id) {
    return this.storage.get(id);
  }

  findAll() {
    return this.storage.getAll();
  }

  save(entity) {
    return this.storage.save(entity);
  }

  delete(id) {
    return this.storage.delete(id);
  }
}

class InMemoryStorage {
  constructor() {
    this.data = new Map();
  }

  get(id) {
    return this.data.get(id);
  }

  getAll() {
    return Array.from(this.data.values());
  }

  save(entity) {
    this.data.set(entity.id, entity);
    return entity;
  }

  delete(id) {
    return this.data.delete(id);
  }
}

const userRepo = new Repository(new InMemoryStorage());

userRepo.save({ id: 1, name: "Alice" });
userRepo.save({ id: 2, name: "Bob" });

console.log(userRepo.findAll());
// [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }]
```

---

## 🏗️ Best Practices

1. **Use patterns appropriately** - Don't over-engineer
2. **Prefer composition** - More flexible than inheritance
3. **Keep it simple** - Choose simplest pattern that solves problem
4. **Document patterns** - Help team understand design
5. **Test thoroughly** - Patterns should make testing easier

---

## 🧪 Interview Questions

### **Q1: What is the Singleton pattern?**

**Answer:** Ensures a class has only one instance and provides global access to it. Useful for configuration, logging, or database connections.

```javascript
class Singleton {
  static #instance;
  static getInstance() {
    if (!this.#instance) {
      this.#instance = new Singleton();
    }
    return this.#instance;
  }
}
```

### **Q2: Explain the Observer pattern.**

**Answer:** Defines one-to-many dependency where observers are notified when subject changes state. Used in event systems, pub/sub, reactive programming.

```javascript
class Subject {
  constructor() {
    this.observers = [];
  }
  notify(data) {
    this.observers.forEach((o) => o.update(data));
  }
}
```

---

## 📚 Key Takeaways

- **Creational**: Singleton, Factory, Builder - object creation
- **Structural**: Module, Decorator, Proxy - object composition
- **Behavioral**: Observer, Strategy, Command - object interaction
- Choose appropriate pattern for problem
- Don't over-engineer - keep it simple
- Patterns make code maintainable and testable

---

**Master design patterns for production-ready code!**
