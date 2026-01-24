# SOLID Principles

> **SOLID design principles in JavaScript**

---

## 🎯 Overview

SOLID is an acronym for five design principles that make software more maintainable, flexible, and scalable. While originally for OOP languages, these principles apply to JavaScript.

---

## 📚 Core Concepts

### **Concept 1: Single Responsibility Principle (SRP)**

**A class should have only one reason to change.**

```javascript
// ❌ Bad - multiple responsibilities
class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }

  save() {
    // Database logic
    const db = connectDatabase();
    db.save(this);
  }

  sendEmail(message) {
    // Email logic
    const mailer = createMailer();
    mailer.send(this.email, message);
  }

  generateReport() {
    // Reporting logic
    return `Report for ${this.name}`;
  }
}

// ✅ Good - single responsibility per class
class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
}

class UserRepository {
  save(user) {
    const db = connectDatabase();
    db.save(user);
  }
}

class EmailService {
  send(email, message) {
    const mailer = createMailer();
    mailer.send(email, message);
  }
}

class ReportGenerator {
  generate(user) {
    return `Report for ${user.name}`;
  }
}
```

### **Concept 2: Open/Closed Principle (OCP)**

**Open for extension, closed for modification.**

```javascript
// ❌ Bad - must modify to add new shapes
class AreaCalculator {
  calculate(shape) {
    if (shape.type === "circle") {
      return Math.PI * shape.radius ** 2;
    } else if (shape.type === "rectangle") {
      return shape.width * shape.height;
    }
    // Must modify this class to add new shapes
  }
}

// ✅ Good - extend without modifying
class Shape {
  getArea() {
    throw new Error("Must implement getArea");
  }
}

class Circle extends Shape {
  constructor(radius) {
    super();
    this.radius = radius;
  }

  getArea() {
    return Math.PI * this.radius ** 2;
  }
}

class Rectangle extends Shape {
  constructor(width, height) {
    super();
    this.width = width;
    this.height = height;
  }

  getArea() {
    return this.width * this.height;
  }
}

class AreaCalculator {
  calculate(shape) {
    return shape.getArea(); // Works for any shape
  }
}

// Add new shape without modifying existing code
class Triangle extends Shape {
  constructor(base, height) {
    super();
    this.base = base;
    this.height = height;
  }

  getArea() {
    return (this.base * this.height) / 2;
  }
}
```

### **Concept 3: Liskov Substitution Principle (LSP)**

**Subtypes must be substitutable for their base types.**

```javascript
// ❌ Bad - violates LSP
class Bird {
  fly() {
    console.log("Flying");
  }
}

class Penguin extends Bird {
  fly() {
    throw new Error("Cannot fly"); // Violates LSP
  }
}

function makeBirdFly(bird) {
  bird.fly(); // Fails for Penguin
}

// ✅ Good - follows LSP
class Bird {
  move() {
    console.log("Moving");
  }
}

class FlyingBird extends Bird {
  move() {
    console.log("Flying");
  }
}

class Penguin extends Bird {
  move() {
    console.log("Swimming");
  }
}

function makeBirdMove(bird) {
  bird.move(); // Works for all birds
}
```

### **Concept 4: Interface Segregation Principle (ISP)**

**Clients shouldn't depend on interfaces they don't use.**

```javascript
// ❌ Bad - fat interface
class Worker {
  work() {}
  eat() {}
  sleep() {}
}

class Robot extends Worker {
  work() {
    console.log("Working");
  }

  eat() {
    // Robots don't eat - unnecessary method
  }

  sleep() {
    // Robots don't sleep - unnecessary method
  }
}

// ✅ Good - segregated interfaces
class Workable {
  work() {}
}

class Eatable {
  eat() {}
}

class Sleepable {
  sleep() {}
}

class Human {
  constructor() {
    Object.assign(this, new Workable(), new Eatable(), new Sleepable());
  }
}

class Robot {
  constructor() {
    Object.assign(this, new Workable()); // Only what it needs
  }
}
```

### **Concept 5: Dependency Inversion Principle (DIP)**

**Depend on abstractions, not concretions.**

```javascript
// ❌ Bad - depends on concrete implementations
class MySQLDatabase {
  save(data) {
    console.log("Saving to MySQL");
  }
}

class UserService {
  constructor() {
    this.database = new MySQLDatabase(); // Tightly coupled
  }

  saveUser(user) {
    this.database.save(user);
  }
}

// ✅ Good - depends on abstraction
class Database {
  save(data) {
    throw new Error("Must implement save");
  }
}

class MySQLDatabase extends Database {
  save(data) {
    console.log("Saving to MySQL", data);
  }
}

class MongoDatabase extends Database {
  save(data) {
    console.log("Saving to MongoDB", data);
  }
}

class UserService {
  constructor(database) {
    // Dependency injection
    this.database = database;
  }

  saveUser(user) {
    this.database.save(user);
  }
}

// Usage
const mysqlService = new UserService(new MySQLDatabase());
const mongoService = new UserService(new MongoDatabase());
```

---

## 🔥 Practical Examples

### **Example 1: SRP in React**

```javascript
// ❌ Bad - component does too much
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // Fetching logic
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then(setUser);
  }, [userId]);

  // Rendering, styling, business logic all mixed
  return (
    <div style={{ padding: 20 }}>
      <h1>{user?.name}</h1>
      <p>{user?.email}</p>
      <button
        onClick={() => {
          // Business logic
          fetch(`/api/users/${userId}`, { method: "DELETE" });
        }}
      >
        Delete
      </button>
    </div>
  );
}

// ✅ Good - separated concerns
function useUser(userId) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);

  return user;
}

function UserProfile({ userId }) {
  const user = useUser(userId);
  const { deleteUser } = useUserActions();

  if (!user) return null;

  return <UserProfileView user={user} onDelete={() => deleteUser(userId)} />;
}
```

### **Example 2: DIP with Dependency Injection**

```javascript
// Logger abstraction
class Logger {
  log(message) {}
}

class ConsoleLogger extends Logger {
  log(message) {
    console.log(message);
  }
}

class FileLogger extends Logger {
  log(message) {
    // Write to file
  }
}

// Service depends on abstraction
class UserService {
  constructor(logger) {
    this.logger = logger;
  }

  createUser(data) {
    // Business logic
    this.logger.log(`User created: ${data.name}`);
  }
}

// Inject dependency
const service = new UserService(new ConsoleLogger());
// Or: new UserService(new FileLogger());
```

---

## 🏗️ Best Practices

1. **SRP: One class, one responsibility**
2. **OCP: Extend behavior without modifying existing code**
3. **LSP: Subtypes must be fully substitutable**
4. **ISP: Small, focused interfaces**
5. **DIP: Inject dependencies, depend on abstractions**

---

## 🧪 Interview Questions

### **Q1: Explain the Single Responsibility Principle.**

**Answer:** A class should have only one reason to change. Each class should focus on one responsibility. Benefits: easier testing, maintenance, and understanding.

### **Q2: What is Dependency Inversion?**

**Answer:** High-level modules shouldn't depend on low-level modules. Both should depend on abstractions. Use dependency injection to provide implementations at runtime.

---

## 📚 Key Takeaways

- **S**ingle Responsibility - One class, one job
- **O**pen/Closed - Extend without modifying
- **L**iskov Substitution - Subtypes are substitutable
- **I**nterface Segregation - Small, focused interfaces
- **D**ependency Inversion - Depend on abstractions

---

**Master solid principles for production-ready code!**
