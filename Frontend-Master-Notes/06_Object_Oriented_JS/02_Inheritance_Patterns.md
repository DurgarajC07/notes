# Inheritance Patterns

> **Prototypal inheritance, class inheritance, and composition**

---

## 🎯 Overview

JavaScript uses prototypal inheritance. Understanding prototype chains, ES6 class inheritance, and composition patterns is crucial for effective object-oriented JavaScript.

---

## 📚 Core Concepts

### **Concept 1: Prototypal Inheritance**

Every object has an internal `[[Prototype]]` link:

```javascript
// Object linking
const animal = {
  eat() {
    console.log("Eating...");
  },
};

const dog = Object.create(animal);
dog.bark = function () {
  console.log("Woof!");
};

dog.eat(); // Eating... (inherited)
dog.bark(); // Woof! (own property)

// Prototype chain
console.log(dog.__proto__ === animal); // true
console.log(animal.__proto__ === Object.prototype); // true
console.log(Object.prototype.__proto__ === null); // true (end of chain)
```

**Constructor Functions and Prototypes:**

```javascript
function Animal(name) {
  this.name = name;
}

Animal.prototype.eat = function () {
  console.log(`${this.name} is eating`);
};

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

const dog = new Dog("Buddy", "Golden Retriever");
dog.eat(); // Buddy is eating
dog.bark(); // Woof!
```

### **Concept 2: ES6 Class Inheritance**

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }

  eat() {
    console.log(`${this.name} is eating`);
  }

  sleep() {
    console.log(`${this.name} is sleeping`);
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name); // Call parent constructor
    this.breed = breed;
  }

  bark() {
    console.log("Woof!");
  }

  // Override parent method
  eat() {
    super.eat(); // Call parent method
    console.log(`${this.breed} loves food!`);
  }
}

const dog = new Dog("Buddy", "Golden Retriever");
dog.eat();
// Buddy is eating
// Golden Retriever loves food!
dog.bark(); // Woof!
dog.sleep(); // Buddy is sleeping
```

### **Concept 3: Composition Over Inheritance**

```javascript
// Composition - mixins
const canEat = {
  eat() {
    console.log(`${this.name} is eating`);
  },
};

const canWalk = {
  walk() {
    console.log(`${this.name} is walking`);
  },
};

const canSwim = {
  swim() {
    console.log(`${this.name} is swimming`);
  },
};

// Create object with multiple behaviors
class Animal {
  constructor(name) {
    this.name = name;
    Object.assign(this, canEat, canWalk);
  }
}

class Fish {
  constructor(name) {
    this.name = name;
    Object.assign(this, canEat, canSwim);
  }
}

const dog = new Animal("Buddy");
dog.eat(); // Buddy is eating
dog.walk(); // Buddy is walking

const fish = new Fish("Nemo");
fish.eat(); // Nemo is eating
fish.swim(); // Nemo is swimming
// fish.walk(); // undefined - doesn't have this method
```

---

## 🔥 Practical Examples

### **Example 1: Method Override**

```javascript
class Shape {
  constructor(color) {
    this.color = color;
  }

  getArea() {
    return 0; // Base implementation
  }

  describe() {
    return `A ${this.color} shape with area ${this.getArea()}`;
  }
}

class Circle extends Shape {
  constructor(color, radius) {
    super(color);
    this.radius = radius;
  }

  getArea() {
    return Math.PI * this.radius ** 2;
  }
}

class Rectangle extends Shape {
  constructor(color, width, height) {
    super(color);
    this.width = width;
    this.height = height;
  }

  getArea() {
    return this.width * this.height;
  }
}

const circle = new Circle("red", 5);
console.log(circle.describe());
// A red shape with area 78.54

const rect = new Rectangle("blue", 4, 5);
console.log(rect.describe());
// A blue shape with area 20
```

### **Example 2: Multiple Inheritance via Mixins**

```javascript
// Mixin functions
const TimestampMixin = (Base) =>
  class extends Base {
    constructor(...args) {
      super(...args);
      this.created = new Date();
    }

    getAge() {
      return Date.now() - this.created;
    }
  };

const ValidationMixin = (Base) =>
  class extends Base {
    validate() {
      return Object.keys(this).every((key) => this[key] != null);
    }
  };

// Apply multiple mixins
class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
}

const EnhancedUser = ValidationMixin(TimestampMixin(User));

const user = new EnhancedUser("Alice", "alice@example.com");
console.log(user.getAge()); // 0 (just created)
console.log(user.validate()); // true
```

### **Example 3: Abstract Base Class Pattern**

```javascript
class AbstractClass {
  constructor() {
    if (new.target === AbstractClass) {
      throw new Error("Cannot instantiate abstract class");
    }
  }

  // Abstract method
  abstractMethod() {
    throw new Error("Must implement abstractMethod");
  }

  // Concrete method
  concreteMethod() {
    return "This is implemented";
  }
}

class ConcreteClass extends AbstractClass {
  abstractMethod() {
    return "Implemented!";
  }
}

// const abstract = new AbstractClass(); // Error
const concrete = new ConcreteClass();
console.log(concrete.abstractMethod()); // Implemented!
console.log(concrete.concreteMethod()); // This is implemented
```

### **Example 4: Delegation Pattern**

```javascript
// Instead of inheritance, delegate to composed objects
class Engine {
  start() {
    console.log("Engine started");
  }

  stop() {
    console.log("Engine stopped");
  }
}

class Wheels {
  rotate() {
    console.log("Wheels rotating");
  }
}

class Car {
  constructor() {
    this.engine = new Engine();
    this.wheels = new Wheels();
  }

  // Delegate to composed objects
  start() {
    this.engine.start();
    this.wheels.rotate();
  }

  stop() {
    this.engine.stop();
  }
}

const car = new Car();
car.start();
// Engine started
// Wheels rotating
```

---

## 🏗️ Best Practices

1. **Prefer composition over inheritance** - More flexible

   ```javascript
   class User {
     constructor() {
       this.permissions = new PermissionManager();
     }
   }
   ```

2. **Keep inheritance hierarchies shallow** - Max 2-3 levels

   ```javascript
   // ✅ Good: shallow
   class Animal {}
   class Dog extends Animal {}

   // ❌ Bad: deep
   class A {}
   class B extends A {}
   class C extends B {}
   class D extends C {}
   ```

3. **Use super() first in constructor** - Required before accessing `this`

   ```javascript
   constructor(name) {
     super(); // Must be first
     this.name = name;
   }
   ```

4. **Avoid modifying prototypes of built-ins** - Can cause issues

   ```javascript
   // ❌ Bad
   Array.prototype.myMethod = function () {};
   ```

5. **Use mixins for cross-cutting concerns** - Timestamp, validation, etc.
   ```javascript
   const Enhanced = TimestampMixin(ValidationMixin(Base));
   ```

---

## 🧪 Interview Questions

### **Q1: Explain prototypal inheritance.**

**Answer:** JavaScript uses prototypal inheritance where objects inherit from other objects via the prototype chain. When accessing a property, JavaScript looks up the chain until found or reaching null.

```javascript
const obj = { a: 1 };
obj.__proto__ === Object.prototype; // true
```

Class inheritance is syntactic sugar over prototypal inheritance.

### **Q2: What's the difference between `__proto__` and `prototype`?**

**Answer:**

- `__proto__` is the actual object used in lookup chain (on instances)
- `prototype` is the object assigned to instances created by constructor (on constructors)

```javascript
function Dog() {}
const dog = new Dog();

dog.__proto__ === Dog.prototype; // true
```

### **Q3: Why prefer composition over inheritance?**

**Answer:** Composition is more flexible:

1. Avoid deep inheritance hierarchies
2. Easier to change behavior at runtime
3. Compose behaviors from multiple sources
4. No "diamond problem"
5. Clearer dependencies

```javascript
// Composition
class User {
  constructor() {
    this.auth = new AuthService();
    this.logger = new Logger();
  }
}
```

---

## 📚 Key Takeaways

- JavaScript uses prototypal inheritance (prototype chains)
- ES6 classes are syntactic sugar over prototypes
- Use `extends` for class inheritance, `super` for parent access
- Prefer composition over inheritance for flexibility
- Keep inheritance hierarchies shallow (2-3 levels max)
- Use mixins for cross-cutting concerns
- Avoid modifying built-in prototypes

---

**Master inheritance patterns for production-ready code!**
