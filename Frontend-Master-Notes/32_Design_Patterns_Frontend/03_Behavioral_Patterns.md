# Behavioral Design Patterns in Frontend Development

## Core Concepts

Behavioral patterns focus on communication between objects, defining how objects interact and distribute responsibility. They help manage complex control flow and object collaboration in frontend applications, making code more flexible and easier to maintain. These patterns are crucial for event handling, state management, and coordinating complex interactions in modern web applications.

## Observer Pattern

### Event Emitter Implementation

```typescript
type EventHandler<T = any> = (data: T) => void;

class EventEmitter<Events extends Record<string, any> = any> {
  private events: Map<keyof Events, Set<EventHandler>> = new Map();

  on<K extends keyof Events>(
    event: K,
    handler: EventHandler<Events[K]>,
  ): () => void {
    if (!this.events.has(event)) {
      this.events.set(event, new Set());
    }

    this.events.get(event)!.add(handler);

    // Return unsubscribe function
    return () => this.off(event, handler);
  }

  off<K extends keyof Events>(
    event: K,
    handler: EventHandler<Events[K]>,
  ): void {
    const handlers = this.events.get(event);
    if (handlers) {
      handlers.delete(handler);
      if (handlers.size === 0) {
        this.events.delete(event);
      }
    }
  }

  emit<K extends keyof Events>(event: K, data: Events[K]): void {
    const handlers = this.events.get(event);
    if (handlers) {
      handlers.forEach((handler) => handler(data));
    }
  }

  once<K extends keyof Events>(
    event: K,
    handler: EventHandler<Events[K]>,
  ): void {
    const onceHandler: EventHandler<Events[K]> = (data) => {
      handler(data);
      this.off(event, onceHandler);
    };
    this.on(event, onceHandler);
  }

  removeAllListeners<K extends keyof Events>(event?: K): void {
    if (event) {
      this.events.delete(event);
    } else {
      this.events.clear();
    }
  }

  listenerCount<K extends keyof Events>(event: K): number {
    return this.events.get(event)?.size ?? 0;
  }
}

// Usage
interface AppEvents {
  userLoggedIn: { userId: string; timestamp: number };
  userLoggedOut: { userId: string };
  dataUpdated: { collection: string; id: string };
  error: { message: string; code: number };
}

const eventBus = new EventEmitter<AppEvents>();

// Subscribe to events
const unsubscribe = eventBus.on("userLoggedIn", (data) => {
  console.log(`User ${data.userId} logged in at ${data.timestamp}`);
});

eventBus.on("userLoggedOut", (data) => {
  console.log(`User ${data.userId} logged out`);
});

// Emit events
eventBus.emit("userLoggedIn", { userId: "123", timestamp: Date.now() });
eventBus.emit("userLoggedOut", { userId: "123" });

// Unsubscribe
unsubscribe();
```

### React State Management with Observer

```typescript
type Listener<T> = (state: T) => void;

class Observable<T> {
  private listeners: Set<Listener<T>> = new Set();

  constructor(private state: T) {}

  getState(): T {
    return this.state;
  }

  setState(newState: T | ((prevState: T) => T)): void {
    this.state = typeof newState === 'function'
      ? (newState as (prevState: T) => T)(this.state)
      : newState;

    this.notify();
  }

  subscribe(listener: Listener<T>): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  private notify(): void {
    this.listeners.forEach(listener => listener(this.state));
  }
}

// Custom hook to use observable in React
function useObservable<T>(observable: Observable<T>): [T, (newState: T | ((prevState: T) => T)) => void] {
  const [state, setState] = React.useState(observable.getState());

  React.useEffect(() => {
    return observable.subscribe(setState);
  }, [observable]);

  return [state, (newState) => observable.setState(newState)];
}

// Usage
interface AppState {
  user: { id: string; name: string } | null;
  theme: 'light' | 'dark';
  notifications: number;
}

const appStore = new Observable<AppState>({
  user: null,
  theme: 'light',
  notifications: 0,
});

const UserProfile: React.FC = () => {
  const [state, setState] = useObservable(appStore);

  const login = () => {
    setState({
      ...state,
      user: { id: '1', name: 'John Doe' },
    });
  };

  return (
    <div>
      {state.user ? (
        <div>Welcome, {state.user.name}!</div>
      ) : (
        <button onClick={login}>Login</button>
      )}
    </div>
  );
};
```

### Pub/Sub Pattern

```typescript
type SubscriptionId = string;

interface Subscription {
  id: SubscriptionId;
  topic: string;
  callback: (data: any) => void;
}

class PubSub {
  private subscriptions: Map<string, Map<SubscriptionId, Subscription>> =
    new Map();
  private nextId = 0;

  subscribe(topic: string, callback: (data: any) => void): SubscriptionId {
    const id = `sub_${this.nextId++}`;

    if (!this.subscriptions.has(topic)) {
      this.subscriptions.set(topic, new Map());
    }

    this.subscriptions.get(topic)!.set(id, { id, topic, callback });

    return id;
  }

  unsubscribe(subscriptionId: SubscriptionId): boolean {
    for (const [topic, subs] of this.subscriptions) {
      if (subs.has(subscriptionId)) {
        subs.delete(subscriptionId);
        if (subs.size === 0) {
          this.subscriptions.delete(topic);
        }
        return true;
      }
    }
    return false;
  }

  publish(topic: string, data: any): void {
    const subs = this.subscriptions.get(topic);
    if (subs) {
      subs.forEach((sub) => {
        try {
          sub.callback(data);
        } catch (error) {
          console.error(`Error in subscription ${sub.id}:`, error);
        }
      });
    }
  }

  publishAsync(topic: string, data: any): Promise<void> {
    return new Promise((resolve) => {
      setTimeout(() => {
        this.publish(topic, data);
        resolve();
      }, 0);
    });
  }

  getTopics(): string[] {
    return Array.from(this.subscriptions.keys());
  }

  getSubscriptionCount(topic?: string): number {
    if (topic) {
      return this.subscriptions.get(topic)?.size ?? 0;
    }
    return Array.from(this.subscriptions.values()).reduce(
      (total, subs) => total + subs.size,
      0,
    );
  }
}

// Global pub/sub instance
const pubsub = new PubSub();

// Usage
const subId1 = pubsub.subscribe("user:login", (data) => {
  console.log("Analytics: User logged in", data);
});

const subId2 = pubsub.subscribe("user:login", (data) => {
  console.log("Notifications: Send welcome message", data);
});

const subId3 = pubsub.subscribe("user:login", (data) => {
  console.log("UI: Update header", data);
});

pubsub.publish("user:login", { userId: "123", timestamp: Date.now() });

// Unsubscribe
pubsub.unsubscribe(subId1);
```

## Strategy Pattern

### Payment Strategy

```typescript
interface PaymentStrategy {
  processPayment(
    amount: number,
  ): Promise<{ success: boolean; transactionId?: string }>;
  validatePaymentDetails(): boolean;
}

class CreditCardStrategy implements PaymentStrategy {
  constructor(
    private cardNumber: string,
    private cvv: string,
    private expiry: string,
  ) {}

  validatePaymentDetails(): boolean {
    // Validate credit card
    return this.cardNumber.length === 16 && this.cvv.length === 3;
  }

  async processPayment(
    amount: number,
  ): Promise<{ success: boolean; transactionId?: string }> {
    if (!this.validatePaymentDetails()) {
      return { success: false };
    }

    console.log(`Processing credit card payment: $${amount}`);
    // Simulate API call
    await new Promise((resolve) => setTimeout(resolve, 1000));

    return {
      success: true,
      transactionId: `cc_${Date.now()}`,
    };
  }
}

class PayPalStrategy implements PaymentStrategy {
  constructor(
    private email: string,
    private password: string,
  ) {}

  validatePaymentDetails(): boolean {
    return this.email.includes("@") && this.password.length >= 8;
  }

  async processPayment(
    amount: number,
  ): Promise<{ success: boolean; transactionId?: string }> {
    if (!this.validatePaymentDetails()) {
      return { success: false };
    }

    console.log(`Processing PayPal payment: $${amount}`);
    await new Promise((resolve) => setTimeout(resolve, 1000));

    return {
      success: true,
      transactionId: `pp_${Date.now()}`,
    };
  }
}

class CryptoStrategy implements PaymentStrategy {
  constructor(private walletAddress: string) {}

  validatePaymentDetails(): boolean {
    return (
      this.walletAddress.startsWith("0x") && this.walletAddress.length === 42
    );
  }

  async processPayment(
    amount: number,
  ): Promise<{ success: boolean; transactionId?: string }> {
    if (!this.validatePaymentDetails()) {
      return { success: false };
    }

    console.log(`Processing crypto payment: $${amount}`);
    await new Promise((resolve) => setTimeout(resolve, 2000));

    return {
      success: true,
      transactionId: `crypto_${Date.now()}`,
    };
  }
}

class PaymentProcessor {
  constructor(private strategy: PaymentStrategy) {}

  setStrategy(strategy: PaymentStrategy): void {
    this.strategy = strategy;
  }

  async process(
    amount: number,
  ): Promise<{ success: boolean; transactionId?: string }> {
    if (!this.strategy.validatePaymentDetails()) {
      throw new Error("Invalid payment details");
    }

    return this.strategy.processPayment(amount);
  }
}

// Usage
const processor = new PaymentProcessor(
  new CreditCardStrategy("4111111111111111", "123", "12/25"),
);

const result1 = await processor.process(99.99);
console.log(result1);

// Switch strategy at runtime
processor.setStrategy(new PayPalStrategy("user@example.com", "password123"));
const result2 = await processor.process(49.99);
console.log(result2);
```

### Sorting Strategy

```typescript
interface SortStrategy<T> {
  sort(items: T[]): T[];
}

class QuickSort<T> implements SortStrategy<T> {
  constructor(private compareFn: (a: T, b: T) => number) {}

  sort(items: T[]): T[] {
    if (items.length <= 1) return items;

    const pivot = items[Math.floor(items.length / 2)];
    const left = items.filter((item) => this.compareFn(item, pivot) < 0);
    const middle = items.filter((item) => this.compareFn(item, pivot) === 0);
    const right = items.filter((item) => this.compareFn(item, pivot) > 0);

    return [...this.sort(left), ...middle, ...this.sort(right)];
  }
}

class BubbleSort<T> implements SortStrategy<T> {
  constructor(private compareFn: (a: T, b: T) => number) {}

  sort(items: T[]): T[] {
    const result = [...items];
    const n = result.length;

    for (let i = 0; i < n - 1; i++) {
      for (let j = 0; j < n - i - 1; j++) {
        if (this.compareFn(result[j], result[j + 1]) > 0) {
          [result[j], result[j + 1]] = [result[j + 1], result[j]];
        }
      }
    }

    return result;
  }
}

class MergeSort<T> implements SortStrategy<T> {
  constructor(private compareFn: (a: T, b: T) => number) {}

  sort(items: T[]): T[] {
    if (items.length <= 1) return items;

    const mid = Math.floor(items.length / 2);
    const left = this.sort(items.slice(0, mid));
    const right = this.sort(items.slice(mid));

    return this.merge(left, right);
  }

  private merge(left: T[], right: T[]): T[] {
    const result: T[] = [];
    let i = 0,
      j = 0;

    while (i < left.length && j < right.length) {
      if (this.compareFn(left[i], right[j]) <= 0) {
        result.push(left[i++]);
      } else {
        result.push(right[j++]);
      }
    }

    return [...result, ...left.slice(i), ...right.slice(j)];
  }
}

// Sorter context
class Sorter<T> {
  constructor(private strategy: SortStrategy<T>) {}

  setStrategy(strategy: SortStrategy<T>): void {
    this.strategy = strategy;
  }

  sort(items: T[]): T[] {
    const start = performance.now();
    const result = this.strategy.sort(items);
    const end = performance.now();
    console.log(`Sort took ${end - start}ms`);
    return result;
  }
}

// Usage
interface User {
  id: number;
  name: string;
  age: number;
}

const users: User[] = [
  { id: 3, name: "Charlie", age: 25 },
  { id: 1, name: "Alice", age: 30 },
  { id: 2, name: "Bob", age: 20 },
];

const compareByAge = (a: User, b: User) => a.age - b.age;
const compareByName = (a: User, b: User) => a.name.localeCompare(b.name);

const sorter = new Sorter(new QuickSort(compareByAge));
const sortedByAge = sorter.sort(users);
console.log(sortedByAge);

sorter.setStrategy(new MergeSort(compareByName));
const sortedByName = sorter.sort(users);
console.log(sortedByName);
```

### Validation Strategy

```typescript
interface ValidationStrategy {
  validate(value: any): { valid: boolean; error?: string };
}

class EmailValidation implements ValidationStrategy {
  validate(value: string): { valid: boolean; error?: string } {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    const valid = emailRegex.test(value);
    return valid ? { valid } : { valid: false, error: "Invalid email format" };
  }
}

class PasswordValidation implements ValidationStrategy {
  constructor(private minLength: number = 8) {}

  validate(value: string): { valid: boolean; error?: string } {
    if (value.length < this.minLength) {
      return {
        valid: false,
        error: `Password must be at least ${this.minLength} characters`,
      };
    }

    if (!/[A-Z]/.test(value)) {
      return { valid: false, error: "Password must contain uppercase letter" };
    }

    if (!/[a-z]/.test(value)) {
      return { valid: false, error: "Password must contain lowercase letter" };
    }

    if (!/[0-9]/.test(value)) {
      return { valid: false, error: "Password must contain number" };
    }

    return { valid: true };
  }
}

class PhoneValidation implements ValidationStrategy {
  validate(value: string): { valid: boolean; error?: string } {
    const phoneRegex = /^\+?[\d\s\-()]+$/;
    const valid =
      phoneRegex.test(value) && value.replace(/\D/g, "").length >= 10;
    return valid ? { valid } : { valid: false, error: "Invalid phone number" };
  }
}

class RangeValidation implements ValidationStrategy {
  constructor(
    private min: number,
    private max: number,
  ) {}

  validate(value: number): { valid: boolean; error?: string } {
    const valid = value >= this.min && value <= this.max;
    return valid
      ? { valid }
      : {
          valid: false,
          error: `Value must be between ${this.min} and ${this.max}`,
        };
  }
}

// Validator context
class FieldValidator {
  constructor(private strategy: ValidationStrategy) {}

  setStrategy(strategy: ValidationStrategy): void {
    this.strategy = strategy;
  }

  validate(value: any): { valid: boolean; error?: string } {
    return this.strategy.validate(value);
  }
}

// Usage in a form
interface FormData {
  email: string;
  password: string;
  phone: string;
  age: number;
}

const validators = {
  email: new FieldValidator(new EmailValidation()),
  password: new FieldValidator(new PasswordValidation(8)),
  phone: new FieldValidator(new PhoneValidation()),
  age: new FieldValidator(new RangeValidation(18, 100)),
};

const formData: FormData = {
  email: "user@example.com",
  password: "Password123",
  phone: "+1234567890",
  age: 25,
};

const errors: Partial<Record<keyof FormData, string>> = {};

Object.entries(formData).forEach(([key, value]) => {
  const validator = validators[key as keyof FormData];
  const result = validator.validate(value);
  if (!result.valid) {
    errors[key as keyof FormData] = result.error;
  }
});

console.log(errors);
```

## Command Pattern

### Undo/Redo System

```typescript
interface Command {
  execute(): void;
  undo(): void;
  redo(): void;
}

class TextEditor {
  private content: string = "";

  getContent(): string {
    return this.content;
  }

  insertText(text: string, position: number): void {
    this.content =
      this.content.slice(0, position) + text + this.content.slice(position);
  }

  deleteText(position: number, length: number): string {
    const deleted = this.content.slice(position, position + length);
    this.content =
      this.content.slice(0, position) + this.content.slice(position + length);
    return deleted;
  }
}

class InsertTextCommand implements Command {
  constructor(
    private editor: TextEditor,
    private text: string,
    private position: number,
  ) {}

  execute(): void {
    this.editor.insertText(this.text, this.position);
  }

  undo(): void {
    this.editor.deleteText(this.position, this.text.length);
  }

  redo(): void {
    this.execute();
  }
}

class DeleteTextCommand implements Command {
  private deletedText: string = "";

  constructor(
    private editor: TextEditor,
    private position: number,
    private length: number,
  ) {}

  execute(): void {
    this.deletedText = this.editor.deleteText(this.position, this.length);
  }

  undo(): void {
    this.editor.insertText(this.deletedText, this.position);
  }

  redo(): void {
    this.execute();
  }
}

class CommandManager {
  private history: Command[] = [];
  private currentIndex: number = -1;

  execute(command: Command): void {
    // Remove commands after current index
    this.history = this.history.slice(0, this.currentIndex + 1);

    command.execute();
    this.history.push(command);
    this.currentIndex++;
  }

  undo(): boolean {
    if (this.currentIndex < 0) return false;

    this.history[this.currentIndex].undo();
    this.currentIndex--;
    return true;
  }

  redo(): boolean {
    if (this.currentIndex >= this.history.length - 1) return false;

    this.currentIndex++;
    this.history[this.currentIndex].redo();
    return true;
  }

  canUndo(): boolean {
    return this.currentIndex >= 0;
  }

  canRedo(): boolean {
    return this.currentIndex < this.history.length - 1;
  }

  getHistory(): string[] {
    return this.history
      .slice(0, this.currentIndex + 1)
      .map((cmd) => cmd.constructor.name);
  }
}

// Usage
const editor = new TextEditor();
const commandManager = new CommandManager();

commandManager.execute(new InsertTextCommand(editor, "Hello", 0));
console.log(editor.getContent()); // "Hello"

commandManager.execute(new InsertTextCommand(editor, " World", 5));
console.log(editor.getContent()); // "Hello World"

commandManager.execute(new DeleteTextCommand(editor, 5, 6));
console.log(editor.getContent()); // "Hello"

commandManager.undo();
console.log(editor.getContent()); // "Hello World"

commandManager.redo();
console.log(editor.getContent()); // "Hello"
```

### Macro Recording

```typescript
class MacroCommand implements Command {
  constructor(private commands: Command[] = []) {}

  addCommand(command: Command): void {
    this.commands.push(command);
  }

  execute(): void {
    this.commands.forEach((cmd) => cmd.execute());
  }

  undo(): void {
    // Undo in reverse order
    for (let i = this.commands.length - 1; i >= 0; i--) {
      this.commands[i].undo();
    }
  }

  redo(): void {
    this.execute();
  }
}

// Canvas drawing commands
class Canvas {
  private elements: Array<{ type: string; props: any }> = [];

  addElement(type: string, props: any): void {
    this.elements.push({ type, props });
  }

  removeElement(index: number): { type: string; props: any } {
    return this.elements.splice(index, 1)[0];
  }

  getElements(): Array<{ type: string; props: any }> {
    return [...this.elements];
  }
}

class DrawCircleCommand implements Command {
  constructor(
    private canvas: Canvas,
    private x: number,
    private y: number,
    private radius: number,
  ) {}

  execute(): void {
    this.canvas.addElement("circle", {
      x: this.x,
      y: this.y,
      radius: this.radius,
    });
  }

  undo(): void {
    const elements = this.canvas.getElements();
    this.canvas.removeElement(elements.length - 1);
  }

  redo(): void {
    this.execute();
  }
}

class DrawRectangleCommand implements Command {
  constructor(
    private canvas: Canvas,
    private x: number,
    private y: number,
    private width: number,
    private height: number,
  ) {}

  execute(): void {
    this.canvas.addElement("rectangle", {
      x: this.x,
      y: this.y,
      width: this.width,
      height: this.height,
    });
  }

  undo(): void {
    const elements = this.canvas.getElements();
    this.canvas.removeElement(elements.length - 1);
  }

  redo(): void {
    this.execute();
  }
}

// Usage - record and replay macros
const canvas = new Canvas();
const manager = new CommandManager();

// Record a macro
const macro = new MacroCommand();
macro.addCommand(new DrawCircleCommand(canvas, 50, 50, 25));
macro.addCommand(new DrawCircleCommand(canvas, 100, 50, 25));
macro.addCommand(new DrawRectangleCommand(canvas, 40, 40, 70, 20));

// Execute macro
manager.execute(macro);
console.log(canvas.getElements());

// Undo entire macro
manager.undo();
console.log(canvas.getElements());

// Redo entire macro
manager.redo();
console.log(canvas.getElements());
```

## State Machine Pattern

### Form Wizard State Machine

```typescript
interface FormState {
  enter(): void;
  next(): FormState | null;
  previous(): FormState | null;
  canProceed(): boolean;
  getData(): any;
  setData(data: any): void;
}

class PersonalInfoState implements FormState {
  private data: { name: string; email: string } = { name: "", email: "" };

  enter(): void {
    console.log("Entering Personal Info step");
  }

  next(): FormState | null {
    if (!this.canProceed()) return null;
    return new AddressState();
  }

  previous(): FormState | null {
    return null; // First step
  }

  canProceed(): boolean {
    return this.data.name.length > 0 && this.data.email.includes("@");
  }

  getData(): any {
    return this.data;
  }

  setData(data: any): void {
    this.data = { ...this.data, ...data };
  }
}

class AddressState implements FormState {
  private data: { street: string; city: string; zip: string } = {
    street: "",
    city: "",
    zip: "",
  };

  enter(): void {
    console.log("Entering Address step");
  }

  next(): FormState | null {
    if (!this.canProceed()) return null;
    return new PaymentState();
  }

  previous(): FormState | null {
    return new PersonalInfoState();
  }

  canProceed(): boolean {
    return (
      this.data.street.length > 0 &&
      this.data.city.length > 0 &&
      this.data.zip.length === 5
    );
  }

  getData(): any {
    return this.data;
  }

  setData(data: any): void {
    this.data = { ...this.data, ...data };
  }
}

class PaymentState implements FormState {
  private data: { cardNumber: string; cvv: string } = {
    cardNumber: "",
    cvv: "",
  };

  enter(): void {
    console.log("Entering Payment step");
  }

  next(): FormState | null {
    if (!this.canProceed()) return null;
    return new ConfirmationState();
  }

  previous(): FormState | null {
    return new AddressState();
  }

  canProceed(): boolean {
    return this.data.cardNumber.length === 16 && this.data.cvv.length === 3;
  }

  getData(): any {
    return this.data;
  }

  setData(data: any): void {
    this.data = { ...this.data, ...data };
  }
}

class ConfirmationState implements FormState {
  enter(): void {
    console.log("Form completed!");
  }

  next(): FormState | null {
    return null; // Last step
  }

  previous(): FormState | null {
    return new PaymentState();
  }

  canProceed(): boolean {
    return true;
  }

  getData(): any {
    return {};
  }

  setData(data: any): void {}
}

class FormWizard {
  private currentState: FormState;
  private allData: any = {};

  constructor() {
    this.currentState = new PersonalInfoState();
    this.currentState.enter();
  }

  next(): boolean {
    if (!this.currentState.canProceed()) {
      console.log("Cannot proceed: validation failed");
      return false;
    }

    this.allData = { ...this.allData, ...this.currentState.getData() };
    const nextState = this.currentState.next();

    if (nextState) {
      this.currentState = nextState;
      this.currentState.enter();
      return true;
    }

    return false;
  }

  previous(): boolean {
    const prevState = this.currentState.previous();

    if (prevState) {
      this.currentState = prevState;
      this.currentState.enter();
      return true;
    }

    return false;
  }

  updateData(data: any): void {
    this.currentState.setData(data);
  }

  getAllData(): any {
    return { ...this.allData, ...this.currentState.getData() };
  }

  canProceed(): boolean {
    return this.currentState.canProceed();
  }
}

// Usage
const wizard = new FormWizard();

wizard.updateData({ name: "John Doe", email: "john@example.com" });
wizard.next(); // Move to address

wizard.updateData({ street: "123 Main St", city: "New York", zip: "10001" });
wizard.next(); // Move to payment

wizard.updateData({ cardNumber: "4111111111111111", cvv: "123" });
wizard.next(); // Move to confirmation

console.log(wizard.getAllData());
```

### Traffic Light State Machine

```typescript
interface TrafficLightState {
  handle(context: TrafficLight): void;
  getColor(): string;
  getDuration(): number;
}

class RedLightState implements TrafficLightState {
  handle(context: TrafficLight): void {
    console.log("Red light - Stop!");
    setTimeout(() => {
      context.setState(new GreenLightState());
    }, this.getDuration());
  }

  getColor(): string {
    return "red";
  }

  getDuration(): number {
    return 5000; // 5 seconds
  }
}

class YellowLightState implements TrafficLightState {
  handle(context: TrafficLight): void {
    console.log("Yellow light - Caution!");
    setTimeout(() => {
      context.setState(new RedLightState());
    }, this.getDuration());
  }

  getColor(): string {
    return "yellow";
  }

  getDuration(): number {
    return 2000; // 2 seconds
  }
}

class GreenLightState implements TrafficLightState {
  handle(context: TrafficLight): void {
    console.log("Green light - Go!");
    setTimeout(() => {
      context.setState(new YellowLightState());
    }, this.getDuration());
  }

  getColor(): string {
    return "green";
  }

  getDuration(): number {
    return 10000; // 10 seconds
  }
}

class TrafficLight {
  private state: TrafficLightState;

  constructor() {
    this.state = new RedLightState();
  }

  setState(state: TrafficLightState): void {
    this.state = state;
    this.state.handle(this);
  }

  getCurrentColor(): string {
    return this.state.getColor();
  }

  start(): void {
    this.state.handle(this);
  }
}

// Usage
const trafficLight = new TrafficLight();
trafficLight.start();
```

## Chain of Responsibility Pattern

### Middleware Chain

```typescript
type NextFunction = () => void;
type MiddlewareFunction = (
  req: Request,
  res: Response,
  next: NextFunction,
) => void;

interface Request {
  method: string;
  url: string;
  headers: Record<string, string>;
  body?: any;
  user?: any;
}

interface Response {
  statusCode: number;
  body: any;
  setStatus(code: number): void;
  json(data: any): void;
}

class ResponseImpl implements Response {
  statusCode: number = 200;
  body: any = null;

  setStatus(code: number): void {
    this.statusCode = code;
  }

  json(data: any): void {
    this.body = data;
  }
}

class MiddlewareChain {
  private middlewares: MiddlewareFunction[] = [];

  use(middleware: MiddlewareFunction): this {
    this.middlewares.push(middleware);
    return this;
  }

  async execute(req: Request, res: Response): Promise<void> {
    let index = 0;

    const next = (): void => {
      if (index < this.middlewares.length) {
        const middleware = this.middlewares[index++];
        middleware(req, res, next);
      }
    };

    next();
  }
}

// Middleware examples
const loggingMiddleware: MiddlewareFunction = (req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
};

const authMiddleware: MiddlewareFunction = (req, res, next) => {
  const token = req.headers["authorization"];

  if (!token) {
    res.setStatus(401);
    res.json({ error: "No authorization token" });
    return;
  }

  // Validate token
  req.user = { id: "123", name: "John Doe" };
  next();
};

const corsMiddleware: MiddlewareFunction = (req, res, next) => {
  res.body = {
    ...res.body,
    headers: {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE",
    },
  };
  next();
};

const rateLimitMiddleware: MiddlewareFunction = (req, res, next) => {
  // Check rate limit
  const rateLimitExceeded = false; // Simulate check

  if (rateLimitExceeded) {
    res.setStatus(429);
    res.json({ error: "Rate limit exceeded" });
    return;
  }

  next();
};

// Usage
const chain = new MiddlewareChain();

chain
  .use(loggingMiddleware)
  .use(corsMiddleware)
  .use(rateLimitMiddleware)
  .use(authMiddleware)
  .use((req, res, next) => {
    // Final handler
    res.json({ message: "Success", user: req.user });
  });

const req: Request = {
  method: "GET",
  url: "/api/users",
  headers: {
    authorization: "Bearer token123",
  },
};

const res = new ResponseImpl();
await chain.execute(req, res);
console.log(res.body);
```

### Validation Chain

```typescript
abstract class ValidationHandler {
  protected next: ValidationHandler | null = null;

  setNext(handler: ValidationHandler): ValidationHandler {
    this.next = handler;
    return handler;
  }

  handle(value: any): { valid: boolean; errors: string[] } {
    const result = this.validate(value);

    if (!result.valid || !this.next) {
      return result;
    }

    const nextResult = this.next.handle(value);
    return {
      valid: result.valid && nextResult.valid,
      errors: [...result.errors, ...nextResult.errors],
    };
  }

  protected abstract validate(value: any): { valid: boolean; errors: string[] };
}

class RequiredValidator extends ValidationHandler {
  protected validate(value: any): { valid: boolean; errors: string[] } {
    const valid = value !== null && value !== undefined && value !== "";
    return {
      valid,
      errors: valid ? [] : ["This field is required"],
    };
  }
}

class MinLengthValidator extends ValidationHandler {
  constructor(private minLength: number) {
    super();
  }

  protected validate(value: string): { valid: boolean; errors: string[] } {
    const valid = value.length >= this.minLength;
    return {
      valid,
      errors: valid ? [] : [`Minimum length is ${this.minLength}`],
    };
  }
}

class MaxLengthValidator extends ValidationHandler {
  constructor(private maxLength: number) {
    super();
  }

  protected validate(value: string): { valid: boolean; errors: string[] } {
    const valid = value.length <= this.maxLength;
    return {
      valid,
      errors: valid ? [] : [`Maximum length is ${this.maxLength}`],
    };
  }
}

class PatternValidator extends ValidationHandler {
  constructor(
    private pattern: RegExp,
    private message: string,
  ) {
    super();
  }

  protected validate(value: string): { valid: boolean; errors: string[] } {
    const valid = this.pattern.test(value);
    return {
      valid,
      errors: valid ? [] : [this.message],
    };
  }
}

// Build validation chain
const usernameValidator = new RequiredValidator();
usernameValidator
  .setNext(new MinLengthValidator(3))
  .setNext(new MaxLengthValidator(20))
  .setNext(
    new PatternValidator(
      /^[a-zA-Z0-9_]+$/,
      "Only letters, numbers, and underscores",
    ),
  );

const passwordValidator = new RequiredValidator();
passwordValidator
  .setNext(new MinLengthValidator(8))
  .setNext(new PatternValidator(/[A-Z]/, "Must contain uppercase letter"))
  .setNext(new PatternValidator(/[a-z]/, "Must contain lowercase letter"))
  .setNext(new PatternValidator(/[0-9]/, "Must contain number"));

// Usage
const usernameResult = usernameValidator.handle("john_doe");
console.log(usernameResult);

const passwordResult = passwordValidator.handle("pass");
console.log(passwordResult);
```

## Mediator Pattern

### Chat Room Mediator

```typescript
interface ChatMediator {
  sendMessage(message: string, from: User, to?: User): void;
  addUser(user: User): void;
  removeUser(user: User): void;
}

class User {
  constructor(
    public id: string,
    public name: string,
    private mediator: ChatMediator,
  ) {}

  send(message: string, to?: User): void {
    console.log(`${this.name} sends: "${message}"`);
    this.mediator.sendMessage(message, this, to);
  }

  receive(message: string, from: User): void {
    console.log(`${this.name} receives from ${from.name}: "${message}"`);
  }
}

class ChatRoom implements ChatMediator {
  private users: Map<string, User> = new Map();

  addUser(user: User): void {
    this.users.set(user.id, user);
    this.broadcast(`${user.name} joined the chat`, user);
  }

  removeUser(user: User): void {
    this.users.delete(user.id);
    this.broadcast(`${user.name} left the chat`, user);
  }

  sendMessage(message: string, from: User, to?: User): void {
    if (to) {
      // Direct message
      to.receive(message, from);
    } else {
      // Broadcast to all except sender
      this.broadcast(message, from);
    }
  }

  private broadcast(message: string, from: User): void {
    this.users.forEach((user) => {
      if (user.id !== from.id) {
        user.receive(message, from);
      }
    });
  }

  getUserCount(): number {
    return this.users.size;
  }
}

// Usage
const chatRoom = new ChatRoom();

const alice = new User("1", "Alice", chatRoom);
const bob = new User("2", "Bob", chatRoom);
const charlie = new User("3", "Charlie", chatRoom);

chatRoom.addUser(alice);
chatRoom.addUser(bob);
chatRoom.addUser(charlie);

alice.send("Hello everyone!");
bob.send("Hi Alice!", alice);
charlie.send("Hey folks!");
```

### Component Communication Mediator

```typescript
interface ComponentMediator {
  notify(sender: Component, event: string, data?: any): void;
  register(component: Component): void;
}

abstract class Component {
  constructor(protected mediator: ComponentMediator) {
    mediator.register(this);
  }

  abstract handleNotification(event: string, data?: any): void;
}

class SearchBox extends Component {
  private query: string = "";

  setQuery(query: string): void {
    this.query = query;
    this.mediator.notify(this, "search", { query });
  }

  handleNotification(event: string, data?: any): void {
    if (event === "clear") {
      this.query = "";
      console.log("SearchBox cleared");
    }
  }
}

class FilterPanel extends Component {
  private filters: Record<string, any> = {};

  setFilter(key: string, value: any): void {
    this.filters[key] = value;
    this.mediator.notify(this, "filter", { filters: this.filters });
  }

  handleNotification(event: string, data?: any): void {
    if (event === "clear") {
      this.filters = {};
      console.log("FilterPanel cleared");
    }
  }
}

class ResultsList extends Component {
  private results: any[] = [];

  handleNotification(event: string, data?: any): void {
    if (event === "search" || event === "filter") {
      console.log("ResultsList updating with:", data);
      this.updateResults(data);
    } else if (event === "clear") {
      this.results = [];
      console.log("ResultsList cleared");
    }
  }

  private updateResults(data: any): void {
    // Fetch and update results based on search and filters
    this.results = [];
  }
}

class PaginationControl extends Component {
  private currentPage: number = 1;

  setPage(page: number): void {
    this.currentPage = page;
    this.mediator.notify(this, "page", { page });
  }

  handleNotification(event: string, data?: any): void {
    if (event === "search" || event === "filter") {
      this.currentPage = 1;
      console.log("PaginationControl reset to page 1");
    } else if (event === "clear") {
      this.currentPage = 1;
      console.log("PaginationControl cleared");
    }
  }
}

class SearchPageMediator implements ComponentMediator {
  private components: Component[] = [];

  register(component: Component): void {
    this.components.push(component);
  }

  notify(sender: Component, event: string, data?: any): void {
    console.log(`Event: ${event}`, data);

    // Notify all components except sender
    this.components.forEach((component) => {
      if (component !== sender) {
        component.handleNotification(event, data);
      }
    });
  }
}

// Usage
const mediator = new SearchPageMediator();

const searchBox = new SearchBox(mediator);
const filterPanel = new FilterPanel(mediator);
const resultsList = new ResultsList(mediator);
const pagination = new PaginationControl(mediator);

searchBox.setQuery("laptop");
filterPanel.setFilter("price", { min: 500, max: 1500 });
pagination.setPage(2);
```

## Real-World Usage

### Complete Shopping Cart with Multiple Patterns

```typescript
// Observer for cart updates
class ShoppingCart extends EventEmitter<{
  itemAdded: { item: CartItem };
  itemRemoved: { itemId: string };
  itemUpdated: { item: CartItem };
  cleared: {};
}> {
  private items: Map<string, CartItem> = new Map();

  addItem(item: CartItem): void {
    this.items.set(item.id, item);
    this.emit("itemAdded", { item });
  }

  removeItem(itemId: string): void {
    this.items.delete(itemId);
    this.emit("itemRemoved", { itemId });
  }

  updateQuantity(itemId: string, quantity: number): void {
    const item = this.items.get(itemId);
    if (item) {
      item.quantity = quantity;
      this.emit("itemUpdated", { item });
    }
  }

  getTotal(): number {
    return Array.from(this.items.values()).reduce(
      (sum, item) => sum + item.price * item.quantity,
      0,
    );
  }

  clear(): void {
    this.items.clear();
    this.emit("cleared", {});
  }
}

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

// Strategy for discount calculation
interface DiscountStrategy {
  calculate(total: number): number;
}

class PercentageDiscount implements DiscountStrategy {
  constructor(private percentage: number) {}

  calculate(total: number): number {
    return total * (this.percentage / 100);
  }
}

class FixedAmountDiscount implements DiscountStrategy {
  constructor(private amount: number) {}

  calculate(total: number): number {
    return Math.min(this.amount, total);
  }
}

// Command for checkout operations
interface CheckoutCommand {
  execute(): Promise<void>;
}

class ApplyDiscountCommand implements CheckoutCommand {
  constructor(
    private cart: ShoppingCart,
    private strategy: DiscountStrategy,
  ) {}

  async execute(): Promise<void> {
    const total = this.cart.getTotal();
    const discount = this.strategy.calculate(total);
    console.log(`Applied discount: $${discount}`);
  }
}

class ProcessPaymentCommand implements CheckoutCommand {
  constructor(
    private cart: ShoppingCart,
    private paymentStrategy: PaymentStrategy,
  ) {}

  async execute(): Promise<void> {
    const total = this.cart.getTotal();
    const result = await this.paymentStrategy.processPayment(total);
    console.log("Payment result:", result);
  }
}

// Usage
const cart = new ShoppingCart();

cart.on("itemAdded", ({ item }) => {
  console.log(`Item added: ${item.name}`);
});

cart.addItem({ id: "1", name: "Laptop", price: 999, quantity: 1 });
cart.addItem({ id: "2", name: "Mouse", price: 29, quantity: 2 });

console.log(`Total: $${cart.getTotal()}`);

// Apply discount
const discountCmd = new ApplyDiscountCommand(cart, new PercentageDiscount(10));
await discountCmd.execute();

// Process payment
const paymentCmd = new ProcessPaymentCommand(
  cart,
  new CreditCardStrategy("4111111111111111", "123", "12/25"),
);
await paymentCmd.execute();
```

## Production Patterns

### Event Bus Singleton

```typescript
class GlobalEventBus {
  private static instance: PubSub;

  static getInstance(): PubSub {
    if (!GlobalEventBus.instance) {
      GlobalEventBus.instance = new PubSub();
    }
    return GlobalEventBus.instance;
  }
}

// Usage across application
const eventBus = GlobalEventBus.getInstance();

// In component A
eventBus.publish("user:login", { userId: "123" });

// In component B
eventBus.subscribe("user:login", (data) => {
  console.log("User logged in:", data);
});
```

## Best Practices

1. **Use Observer for loose coupling** - Components don't need to know about each other
2. **Strategy for algorithm families** - Swap implementations at runtime
3. **Command for undo/redo** - Encapsulate operations as objects
4. **State Machine for complex workflows** - Clear state transitions
5. **Chain of Responsibility for middleware** - Request processing pipeline
6. **Mediator for complex interactions** - Centralize communication logic
7. **Event naming conventions** - Use consistent event names (user:login, not userLogin)
8. **Unsubscribe properly** - Prevent memory leaks in observers
9. **Immutable state in stores** - Prevent unintended mutations
10. **Type-safe events** - Use TypeScript for event types and payloads

## 10 Key Takeaways

1. **Observer Pattern** enables loose coupling through event-driven communication
2. **Strategy Pattern** allows runtime algorithm selection without conditionals
3. **Command Pattern** encapsulates operations for undo/redo and macro recording
4. **State Machine** manages complex state transitions in workflows and UI
5. **Chain of Responsibility** creates processing pipelines for middleware
6. **Mediator Pattern** centralizes complex object interactions
7. **Pub/Sub** is Observer with topics, great for cross-component communication
8. **Behavioral patterns** define how objects collaborate and communicate
9. **Event-driven architecture** is built on Observer and Mediator patterns
10. **Choose based on problem** - Observer for notifications, Strategy for algorithms, Command for operations
